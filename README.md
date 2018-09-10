# 인프라 구축 히스토리
## 1. AWS에 Kubernetes 배포
### kops
[kops](https://github.com/kubernetes/kops)는 kubeadm를 활용한 k8s 클러스터 구성 및 관리 도구입니다. [AWS용 가이드](https://github.com/kubernetes/kops/blob/master/docs/aws.md)를 참고하여 사전 조건을 완료합니다.
- kubectl, kops CLI 설치
- kops용 액세스키 생성 및 권한 부여 (AWS IAM)
- 버저닝이 활성화된 kops용 클러스터 상태 저장소 생성 (AWS S3)
- k8s API 도메인용 DNS 호스트존 구성 (AWS Route53)

설치 및 관리를 위해 아래 환경변수들을 로그인 셸 부팅 스크립트에 복사합니다.
```
export AWS_ACCESS_KEY_ID=<액세스키>
export AWS_SECRET_ACCESS_KEY=<시크릿키>
export NAME=k8s.strix.kr # 추후 k8s API 엔드포인트가 api.k8s.strix.kr로 설정됩니다.
export KOPS_STATE_STORE=s3://$NAME # s3 버킷 도메인을 동일하게 구성했습니다.
export REGION=ap-northeast-2
export ZONES=ap-northeast-2a
```

클러스터를 생성합니다.
```
$ kops create cluster \
  # private VPC로 클러스터를 구성하도록 합니다.
  --topology private \
  # 마스터 및 워커 노드들로의 ssh 연결을 포워딩하는 bastion 인스턴스를 생성하도록 합니다.
  --bastion \
  # 리전 및 존
  --zones $ZONES \
  --master-zones $ZONES \
  # EC2 티어
  --master-size c4.large \
  --node-size m4.large \
  # 초기 워커 노드 갯수
  --node-count 2 \
  # CNI; Container Network Interface 구성
  --networking kube-router \
  --name $NAME
```

- k8s API 엔드포인트:
  - api.k8s.strix.kr
- SSH 접속:
  - 공개키는 `~/.ssh/id_rsa.pub`에 위치합니다.
  - `ssh-add ~/.ssh/id_rsa` 로컬 SSH Agent에 개인키를 등록합니다.
  - 최초 bastion 노드에 접속한 뒤에 `ssh admin@<anynode>`로 VPC 내의 노드로 접속 할 수 있습니다.
  - `ssh -A -i ~/.ssh/id_rsa admin@bastion.k8s.strix.kr`
  - admin으로 접속 후에 root 계정을 이용 할 수 있습니다.
  - root 패스워드는 설정하지 않았으며 `sudo su`로 로그인합니다.
- kubectl config:
  - cluster, context, user가 `~/.kube/config`에 생성됩니다.
  - 다른 머신에서 kubectl config를 생성하려는 경우에는 해당 머신에서 kops 설정을 마친 후 `kops export kubecfg --name $NAME --state $KOPS_STATE_STORE`를 실행하세요.
- 클러스터 검증:
  - DNS에 자동으로 등록된 k8s.strix.kr 레코드를 확인 할 수 있습니다.
  - `kops validate cluster`
  - `kubectl get nodes`
- 기본적인 클러스터 구성 변경 및 업그레이드 가이드:
  https://github.com/kubernetes/kops/blob/master/docs/cli/kops_rolling-update.md

### 문제 해결
1. [롤링 업데이트](https://github.com/kubernetes/kops/blob/master/docs/cli/kops_rolling-update.md) 도중 예기치 못한 이유로 노드 방출에 실패하여 클러스터가 마비되는 경우.
```
$ kops validate cluster
$ kubectl describe node
$ kubectl cluster-info dump
$ watch kubectl get deploy --all-namespaces
$ watch kubectl get pod --all-namespaces
```
위 명령어 등으로 문제점을 찾습니다. 만약 특정 pod을 제거하지못하는 경우 아래 명령어로 pod을 제거하여 수동으로 노드를 방출하도록 도울 수 있습니다.
```
$ kubectl delete pod <셀렉터> --grace-period=0 --force
```
문제가 되는 pod이 많은 경우 아래 명령어로 다수의 노드를 제거하는 명령을 출력 할 수 있습니다.
```
$ kubectl get pods --all-namespaces -o wide | grep <키워드> | awk '{print "kubectl delete pod", $2, "-n", $1, "--force --grace-period=0"}'
```
kubectl이나 kops가 마비되는 경우, DNS를 확인하고 수동으로 api.k8s.strix.kr 을 API 엔드포인트에 연결하세요.

외부 k8s 대시보드 엔드포인트가 마비되는 경우엔 프록시를 열고 로컬에서 관리자의 service account 토큰으로 접속합니다. 이 때 토큰에 대해서는 아래에서 설명합니다.
```
$ kubectl proxy
$ open http://localhost:8001/api/v1/namespaces/kube-system/services/https:kube-dashboard:/proxy/
```

## 2. DNS 서비스 제공자 설정
클러스터 생성 후 kops를 이용해서 클러스터의 DNS 서비스 제공자를 [CoreDNS](https://coredns.io/plugins/kubernetes/)로 [변경](https://github.com/kubernetes/kops/blob/master/docs/cluster_spec.md#kubedns)하고 클러스터를 [업데이트](https://github.com/kubernetes/kops/blob/master/docs/changing_configuration.md)합니다. 이 때 롤링 업데이트는 필요하지 않습니다.

클러스터 생성시 기본 DNS 서비스 제공자는 **kube-dns**로 지정되어 있습니다. 클러스터에 **coredns**가 배포된 이후 kube-dns를 제거하고 dns-auto-scaler 애드온의 배포 커맨드를 수정합니다.

```
$ kubectl delete --namespace=kube-system deployment kube-dns
$ kubectl patch deployment/kube-dns-autoscaler -n kube-system --type json -p '[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/command",
    "value": [
              "/cluster-proportional-autoscaler",
              "--namespace=kube-system",
              "--configmap=kube-dns-autoscaler",
              "--target=deployment/coredns",
              "--default-params={\"linear\":{\"coresPerReplica\":256,\"nodesPerReplica\":16,\"preventSinglePointFailure\":true}}",
              "--logtostderr=true",
              "--v=2"
            ]
  }
]'
```

coredns는 kube-dns에 비해 DNS 서비스를 구성 할 수 있는 다양한 기능을 제공하며, 그 구성을 ConfigMap 리소스를 통해 손쉽게 [편집](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#coredns-configmap-options) 할 수 있습니다.
  - [DNS 포워딩](https://coredns.io/plugins/proxy/):
    타 네임서버로 쿼리 요청을 위임할 수 있습니다.
  - [DNS 페더레이션](https://coredns.io/plugins/federation/):
    클러스터 레벨의 결합이 필요한 경우 클러스터간의 서비스 탐지를 결합 할 수 있습니다.
  - [기타 플러그인들](https://coredns.io/plugins/)

이후 k8s 클러스터 페더페이션이 필요한 경우 coreos의 DNS 페더레이션을 플러그인을 통해 해결 할 수 있습니다. 현 시점에서는 coredns의 기본 설정에 아래의 설정만을 반영합니다.
  - k8s 내부 DNS 서비스의 upstream 네임서버를 Google의 로컬 DNS 서버(8.8.8.8, 8.8.4.4)로 변경합니다.
    - 긴급 상황에서 [로컬 DNS 서버의 cache를 직접 flush](https://developers.google.com/speed/public-dns/cache)하여 DNS 구성의 문제를 확인 할 수 있습니다.

ConfigMap의 coredns 설정을 업데이트하고 coredns의 pod들에 수동으로 리로드 시그널(`kill -SIGUSR1 1`)을 보냅니다.
  - ConfigMap이 업데이트되었을 때 coredns가 자동으로 DNS 구성을 리로드하지 않습니다.
  - 잘못된 설정으로 DNS가 죽어버리면 설정을 업데이트하고 coredns의 pod들을 모두 재생성하면 복구 할 수 있습니다.

```
$ kubectl apply -f ./0-coredns-config.yaml

# 설정을 리로드
$ kubectl get pods -l k8s-app=kube-dns -n kube-system | awk 'NR>1{print "kubectl exec -n kube-system ", $1, "-- kill -SIGUSR1 1"}' | bash

# 재생성
$ kubectl delete pod -l k8s-app=kube-dns -n kube-system
```


## 3. 네트워크 정책 설정
[kube-router](https://github.com/cloudnativelabs/kube-router)는 k8s 클러스터에 CNI 레이어를 담당하는 애드온으로 설치되었습니다. 현 시점에서 k8s 클러스터의 기본 구성은 k8s network policy API의 스펙을 실제로 구현하지 않고 있습니다.

이 때문에 클러스터에 namespace/pod/ip/port 등을 기준으로 inbound/outbound 방화벽 정책을 설정하려면 이 같은 [별도의 애드온](https://github.com/kubernetes/kops/blob/master/docs/networking.md)이 필요합니다.

### 네임스페이스간 방화벽
기본 네임스페이스 **default** 외에 추가로 **prod**, **dev** 네임스페이스를 생성합니다. (ref. **1-namespaces**) 네임스페이스 레벨의 방화벽을 적용하기 위해서 각 네임스페이스에 다음과 같이 레이블링합니다. (ref. **1-set-common-namespaces**)
- env=common:
  - default
  - kube-system
  - kube-public
- env=prod
  - prod
- env=dev
  - dev

레이블을 기준으로 아래의 다이어그램은 적용된 네임스페이스 레이블간 허용 트래픽을 보여줍니다. (ref. **2-namespace-network-policies.yaml**)
```
env=prod <-> env=common <-> env=dev
```

현 시점(18.09.01)에서 네트워크 정책 리소스 업데이트시 kube-router가 [networking.k8s.io/NetworkPolicy API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#networkpolicy-v1-networking-k8s-io) 스펙을 지원하는 점에 유의해야합니다. 또한 `namespaceSelector.matchExpressions`가 구현되지 않았기에 `namespaceSelector.matchLabels`를 이용합니다.

### 4. 네임스페이스간 방화벽 테스트 도구
prod, default, dev의 각 네임스페이스에 **busybox**라는 이름으로 service/deployment를 등록했습니다. (ref. **3-busyboxes**)

배포된 pod들은 네트워크 정책 디버깅 용도로 생성되었습니다. 작성된 스크립트(**ref. 4-busyboxes-test-***)를 이용하면 손쉽게 트래픽을 점검 할 수 있습니다.

현재 방화벽 정책에 대한 올바른 테스트 결과입니다.
```
dehypnosis-mac:k8s dehypnosis$ ./4-busyboxes-test default dev
 * Send HTTP request [default -> dev]
 * ip-172-20-36-62.ap-northeast-2.compute.internal/default/busybox-57c9c54c8-gxrrl -> busybox.dev
<xmp>
Hello World


                                       #         .
                                 # # #        ==
                              # # # # #    ===
                           /""""""""""""""""\___/ ===
                      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
                           \______ o          _,/
                            \      \       _,'
                             `'--.._\..--''
</xmp>
Connecting to busybox.dev (100.71.202.68:80)
-                    100% |*******************************|   439   0:00:00 ETA


dehypnosis-mac:k8s dehypnosis$ ./4-busyboxes-test default prod
 * Send HTTP request [default -> prod]
 * ip-172-20-36-62.ap-northeast-2.compute.internal/default/busybox-57c9c54c8-gxrrl -> busybox.prod
Connecting to busybox.prod (100.64.217.8:80)
<xmp>
Hello World


                                       #         .
                                 # # #        ==
                              # # # # #    ===
                           /""""""""""""""""\___/ ===
                      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
                           \______ o          _,/
                            \      \       _,'
                             `'--.._\..--''
</xmp>
-                    100% |*******************************|   439   0:00:00 ETA


dehypnosis-mac:k8s dehypnosis$ ./4-busyboxes-test prod default
 * Send HTTP request [prod -> default]
 * ip-172-20-36-62.ap-northeast-2.compute.internal/prod/busybox-57c9c54c8-r6xjs -> busybox.default
<xmp>
Hello World


                                       #         .
                                 # # #        ==
                              # # # # #    ===
                           /""""""""""""""""\___/ ===
                      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
                           \______ o          _,/
                            \      \       _,'
                             `'--.._\..--''
</xmp>
Connecting to busybox.default (100.64.158.209:80)
-                    100% |*******************************|   439   0:00:00 ETA


dehypnosis-mac:k8s dehypnosis$ ./4-busyboxes-test prod dev
 * Send HTTP request [prod -> dev]
 * ip-172-20-36-62.ap-northeast-2.compute.internal/prod/busybox-57c9c54c8-r6xjs -> busybox.dev
Connecting to busybox.dev (100.71.202.68:80)
wget: can't connect to remote host (100.71.202.68): Connection refused
command terminated with exit code 1

dehypnosis-mac:k8s dehypnosis$ ./4-busyboxes-test dev prod
 * Send HTTP request [dev -> prod]
 * ip-172-20-50-69.ap-northeast-2.compute.internal/dev/busybox-57c9c54c8-j9gsc -> busybox.prod
Connecting to busybox.prod (100.64.217.8:80)
wget: can't connect to remote host (100.64.217.8): Connection refused
command terminated with exit code 1

dehypnosis-mac:k8s dehypnosis$ ./4-busyboxes-test dev default
 * Send HTTP request [dev -> default]
 * ip-172-20-50-69.ap-northeast-2.compute.internal/dev/busybox-57c9c54c8-j9gsc -> busybox.default
<xmp>
Hello World


                                       #         .
                                 # # #        ==
                              # # # # #    ===
                           /""""""""""""""""\___/ ===
                      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
                           \______ o          _,/
                            \      \       _,'
                             `'--.._\..--''
</xmp>
Connecting to busybox.default (100.64.158.209:80)
-                    100% |*******************************|   439   0:00:00 ETA
```

더 세부적인 디버깅을 위해서 [kube-router가 제공하는 툴 박스](https://github.com/cloudnativelabs/kube-router/blob/master/docs/pod-toolbox.md#pod-toolbox)를 이용 할 수 있습니다.

```
$ KR_POD=$(basename $(kubectl -n kube-system get pods -l k8s-app=kube-router --output name|head -n1))
$ kubectl -n kube-system exec -it ${KR_POD} bash
```

## 5. AWS VPC 피어링 (외부 자원 연결)
이 작업으로 추후 각종 AWS RDS(및 각종 AWS 자원)의 사용에서 헤어핀 트래픽(U자형)으로 인한 지연 시간과 네트워크 비용을 감소시킬 수 있습니다.

k8s가 위치한 EC2 인스턴스의 VPC(k8s.strix.kr VPC)와 RDS 인스턴스의 VPC(db.k8s.strix.kr VPC)를 [피어링](https://docs.aws.amazon.com/ko_kr/AmazonVPC/latest/PeeringGuide/vpc-peering-basics.html)합니다.

- RDS 인스턴스에 할당된 도메인을 소유한 DNS 존에 CNAME 레코드로 연결하고 이후에는 생성한 레코드의 도메인으로 접근합니다.
- 인터넷 접근을 허용하려면 RDS 인스턴스의 인터넷 접근을 허용하고, 제한된 public CIDR/IP 내에서 inbound를 허용하는 보안 그룹(현 시점에서 **internet-db.k8s.strix.kr**로 구성)을 추가로 적용합니다.
  - 인터넷 접근이 활성화된 RDS에 할당된 도메인은 VPC 내부에서는 private IP로 외부에서는 public IP로 해석됩니다.
- k8s VPC 내에서 private IP로의 접근을 허용하려면 private inbound를 허용하는 보안 그룹(현 시점에서 **peering-db.k8s.strix.kr**로 구성)을 적용합니다. 이를 통해 헤어핀 트래픽을 방지합니다.

이렇게 RDS가 할당한 도메인을 다시 한번 소유한 DNS 존의 CNAME 레코드로 연결하면 (ex. my.db.k8s.strix.kr -> blabla-blabla.blabla.ap-northeast-2.rds.amazonaws.com), 추후 RDS 엔드포인트의 변경에 빠르게 대응 할 수 있습니다.


## 6. 패키지 설치 도구

### helm 및 tiler/kubeapps 설치
kubectl와 yaml 파일들로 소프트웨어 패키지를 관리 할 수 있지만, 복잡한 소프트웨어를 클러스터에 설치하고 관리하는 일을 더 쉽게하기 위해 [helm](https://github.com/helm/helm)  패키지 매니저를 사용합니다.

로컬과 클러스터에 각각 client(helm), server(tiler)를 초기화합니다. 추후 다른 머신에서는 로컬의 helm만 초기화하게 됩니다.
```
$ helm init
```

다음으로 tiler에게 service account를 생성해주고 클러스터에 전체에 관한 권한을 위임합니다. 또한 helm chart(패키지) 관리를 돕는 웹 서비스 [kubeapps](https://github.com/kubeapps/kubeapps)를 설치합니다. (ref. **5-install-tiler-and-kube-apps**)

### kubeapps를 프록시하여 접속
지금까지는 관리자가 kubectl을 통해서 k8s API를 이용 할 땐 자동으로 구성된 HTTP Basic Auth 방식의 인증을 사용했습니다. (kubectl config view로 확인)

당장 kubeapps와 같은 웹 서비스에 k8s 관리자 권한을 위임 할 때는 service account 리소스를 생성하여 토큰 인증 방식을 이용합니다.

차후에 통합 IAM 시스템이 구성되고 k8s API와 연동되면, service account나 HTTP Basic Auth 방식은 장애 대응 이외의 경우에는 사용하지 않습니다.

여기서 생성한 service account의 토큰은 이후에 k8s 대시보드 같은 k8s API의 cluster-admin 권한이 필요한 모든 서비스에 접근 할 때도 이용 할 수 있습니다. (ref. **6-admin-service-token**)

```
$ kubectl get secret $(kubectl get sa admin -o jsonpath='{.secrets[].name}') -o jsonpath='{.data.token}' | base64 --decode; echo
```
위처럼 관리자 service account 토큰을 확인 할 수 있습니다.

```
$ open http://127.0.0.1:8080
$ kubectl port-forward --namespace kubeapps svc/kubeapps 8080:80
```
당장 웹 서버가 없기에 kubectl port-forward를 통해 로컬에서 접속합니다.

## 7. 설치된 패키지
- [default/cert-manager](https://github.com/jetstack/cert-manager):
  Ingress 리소스에 관련 annotation을 추가하면 ACME 프로토콜을 통해 Let's Encrypt 등의 CA에게 인증서를 요청, 적용, 갱신하는 작업을 자동화합니다.
    - 설치시 annotation-shim 기능을 미리 활성화합니다.
    - 설치 후 AWS Route53 제어 권한을 갖는 IAM 액세스키를 생성하고, **letsencrypt** ClusterIssuer 리소스를 생성합니다. (ref. **7-cert-issuer.yaml**)

- [default/nginx-ingress](https://github.com/kubernetes/ingress-nginx):
  Ingress 리소스의 업데이트를 해석하여 로드밸런서에 연결합니다. 이를 통해 내부 Service를 Ingress와 연결하여 인터넷 게이트웨이에 연결 할 수 있습니다.
    - 사용하려는 주 도메인에 로드밸런서를 연결하고, 와일드카드 서브 도메인(*)에 주 도메인을 CNAME으로 연결합니다. AWS Route53에 호스팅하는 다른 도메인도 같은 방식으로 리버스 프록시를 자동화 할 수 있습니다.
    - kubeapps-dashboard 서비스를 연결하는 Ingress를 생성합니다. 이때 tls-acme annotation을 통해 TLS 인증서 발급을 자동화합니다. (ref. **8-kubeapps-ingress.yaml**)
    - `kubectl logs -l app=nginx-ingress -n default`로 nginx 구성의 업데이트 진행을 확인 할 수 있습니다. 수 초내로 완료됩니다.
    - `kubectl logs -l app=cert-manager -n default`로 인증서 발급 진행을 확인 할 수 있습니다. 수 분내로 완료됩니다.
    - 이제 https://apps.k8s.strix.kr 로 접속 할 수 있습니다.

- [kube-system/kube-dashboard](https://github.com/kubernetes/dashboard):
  대시보드는 k8s 클러스터에 등록된 모든 리소스에 대한 명세를 확인하고 수정 할 수 있는 웹 UI를 제공합니다. 또한 클러스터 node 및 각 pod들이 소비하는 CPU/RAM에 대한 메트릭을 제공합니다. 또한 웹 상에서 각 pod에서 수집된 로그를 확인하고 shell을 구동 할 수 있습니다. (ref. **9-kube-dashboard-ingress.yaml**)
    - kube-dashboard 서비스를 연결 하는 Ingress를 생성합니다. 마찬가지로 관리자 service account 토큰으로 접속합니다. 이제 https://k8s.strix.kr 로 접속 할 수 있습니다.

- [kube-system/kube-backup](https://github.com/pieterlange/kube-backup):
  주기적으로 모든 리소스의 스펙을 특정 git 저장소에 백업하는 배치 프로세스를 생성합니다. (ref. **10-kube-backup.yaml**)
    - Secret 리소스는 [git-crypt](https://github.com/AGWA/git-crypt)로 대칭키 암호화되어 커밋됩니다.
    - helm 차트가 제공되지 않으므로 수동으로 설치되었습니다.

- [default/dokuwiki](https://www.dokuwiki.org/ko:dokuwiki):
  오픈소스 위키 소프트웨어입니다. (ref. **11-dokuwiki-ingress.yaml**)
    - dokuwiki 서비스를 연결하는 Ingress를 생성합니다. 이제 https://wiki.strix.kr 로 접속 할 수 있습니다.
    
- [default/keycloak](https://www.keycloak.org):
  오픈소스 통합 IAM 소프트웨어입니다. 이 통합 IAM 서비스에 RBAC 체계로 지금까지 등록한 내부 서비스의 접근을 제어하게됩니다. 또한 외부 서비스의 인증 및 접근 제어 역시 통합 IAM에 등록되어 관리됩니다. (ref. **12-keycloak-secret-pvc-ingress.yaml**)
    - 본 서비스는 핵심적인 production 데이터를 관리하기에 데이터베이스 구성 및 관리를 AWS RDS에 위임합니다. AWS RDS에서 Postgres 데이터베이스를 피어링된 VPC 안에 구성하고 도메인을 할당하여, 설치시 helm 차트의 옵션에 반영해줍니다.
    - 추가로 데이터베이스 접근 패스워드를 Secret에서 읽을 수 있도록 keycloak-postgres-auth Secret을 생성하고, 설치시 helm 차트의 옵션에 반영해줍니다.
    - 추가로 UI 테마를 변경 할 수 있도록 Persistent Volume을 생성하기 위해서 별도의 PVC를 생성하고, 설치시 helm 차트의 옵션에 [반영](https://github.com/helm/charts/tree/master/stable/keycloak#providing-a-custom-theme)해줍니다.
    - 설치 완료 후에 외부에 연결 할 수 있도록 Ingress를 생성해줍니다. 이제 https://iam.k8s.strix.kr 로 접속 할 수 있습니다.

## 8. 통합 IAM 적용

### realm 및 role 생성
keycloak 서비스에 접속하여 목적에 맞게 realm과 role을 생성합니다.
- **master** realm
  - 기본으로 생성되어 있는 realm이며 k8s API, k8s dashboard, kubeapps 등 사내 서비스들에 대한 인증과 접근 제어를 담당합니다. 현시점에서 아래와 같은 역할을 갖습니다.
    - **admin** role (기본): IAM 전체 시스템에 대한 관리 권한을 갖습니다.
    - **operator** role: prod/dev realm에 대한 관리 권한을 갖습니다.
    - **developer** role: dev realm에 대한 관리 권한을 갖습니다.
    - **manager** role: prod/dev realm의 모든 리소스를 조회하고 유저를 관리 할 수 있습니다.
  - 역할 관리의 편의를 위해 동일 한 이름의 그룹(group)을 생성하고 위에서 생성한 역할을 그룹에 맵핑합니다.
  - 관리자 콘솔의 보안을 향상시키기 위해서 추후 OTP 인증 등을 활성화 할 수 있습니다.
- **prod 및 dev** realm
  - 추후 배포 환경별로 엔드유저들의 인증과 접근 제어를 담당합니다. 현시점에서는 아무 역할도 등록되지 않았습니다.
  - 보안을 위해 기본으로 제공되는 클라이언트(keycloak 관리자 콘솔 등)를 모두 비활성화 합니다.
  
### k8s API 연동
통합 IAM의 인증 서비스를 k8s API 및 대시보드와 연동합니다.

#### k8s RBAC 구성
[k8s의 RBAC 시스템](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)은 기본적으로 활성화되어 있습니다. 유저별로 리소스 제어 권한을 분리하여 관리하기 위해서 아래의 그룹(k8s Group)에 역할(k8s ClusterRole)을 바인딩합니다.

이 때 직관적인 이해를 위해서 keycloak의 역할과 동일한 이름의 그룹에 역할을 바인딩합니다. 그룹(k8s Group) 자체는 명시적으로 생성하는 리소스가 아니므로 롤 바인딩(k8s ClusterRoleBinding 및 RoleBinding) 리소스만 생성하면 됩니다..

- **operator** group:
  - k8s 클러스터 리소스의 모든 관리 역할(cluster-admin ClusterRole)을 바인딩합니다.
- **developer** group:
  - dev 네임스페이스 리소스의 모든 관리 역할(admin ClusterRole)을 바인딩합니다.
  - prod 및 default 네임스페이스의 열람 역할(view ClusterRole)을 바인딩합니다.
  - 특정 클러스터 리소스(namespaces, nodes, persistent volumes, roles, storage classses)의 열람 역할(cluster-view ClusterRole)을 바인딩합니다.

#### k8s API를 통합 IAM에 연동
지금까지 k8s API의 인증에 admin service account의 액세스 토큰, 또는 admin 계정의 HTTP Basic Auth 방식을 사용했습니다.

k8s API의 인증과 RBAC을 통합 IAM에 연동합니다.

먼저 keycloak master realm에 **kubernetes** 클라이언트를 등록합니다.
  - Standard Flow 인증(OpenID Connect)을 활성화합니다.
  - AcceccType을 Confidential로 두어 클라이언트 secret키를 생성합니다.
  - Valid Redirect URLs에 http://localhost:8000/ 을 추가합니다. 이는 추후 로컬에서 kubectl 인증 설정을 업데이트 하는데 사용될 URL입니다.
  - 클라이언트 role을 생성하고 각각 기존의 realm role에 연결시킵니다.
    - **operator** role: k8s의 operator 그룹으로 맵핑될 역할입니다.
    - **developer** role: k8s의 developer 그룹으로 맵핑될 역할입니다.
  - 클라이언트 role들을 토큰의 **groups** 클레임에 포함하는 맵퍼를 추가 합니다.
    - 참고) 클레임 크기를 줄이기 위해서 Full Scope Access를 비활성화하면 맵퍼가 동작하지 않는 [버그](https://issues.jboss.org/browse/KEYCLOAK-5259)가 있습니다.

다음으로 [k8s 클러스터에 OIDC 인증](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)을 활성화합니다. kops를 이용해 클러스터 스펙에 OIDC 인증을 추가하고, 업데이트 후 마스터 노드의 롤링 업데이트가 필요합니다.
```
$ kops edit cluster
```
```
spec:
  kubeAPIServer:
    oidcClientID: kubernetes
    oidcGroupsClaim: groups
    oidcIssuerURL: https://iam.k8s.strix.kr/realms/master
```

```
$ kops update cluster --yes
$ kops rolling-update cluster --yes
```


#### kubectl 설정
롤링 업데이트가 끝나면 이제 OIDC 인증이 활성화되었습니다. kubect config에 OIDC 인증 설정을 추가 구성합니다. 

```
$ kubectl config set-credentials k8s.strix.kr-oidc \
  --auth-provider oidc \
  --auth-provider-arg idp-issuer-url=https://iam.k8s.strix.kr/realms/master \
  --auth-provider-arg client-id=kubernetes \
  --auth-provider-arg client-secret=<keycloak에서 발급된 클라이언트 시크릿>

$ kubectl config set-context k8s.strix.kr-oidc \
  --namespace dev \
  --cluster=k8s.strix.kr --user=k8s.strix.kr-oidc
```

이제 기존 관리자 인증과 새로운 OIDC 인증을 모두 이용 할 수 있습니다. OIDC 인증으로 설정을 전환합니다.
```
$ kubectl config use-context k8s.strix.kr-oidc
Switched to context "k8s.strix.kr-oidc".

$ kubectl get pods
Unable to connect to the server: No valid id-token, and cannot refresh without refresh-token
```

인증은 적절히 구성되었습니다. 하지만 아직 액세스 토큰이 발급되지 않았습니다. ID 토큰 및 refresh 토큰을 받급 받아 설정 파일에 반영해주는 도우미 프로그램 [kubelogin](https://github.com/int128/kubelogin/releases)을 설치합니다.
```
$ wget https://github.com/int128/kubelogin/releases/download/1.5/kubelogin_darwin_amd64
$ mv kubelogin_darwin_amd64 /usr/local/bin/kubelogin
```

이제 모든 준비가 완료되었습니다.
```
$ kubelogin
2018/09/04 18:34:28 Reading /Users/daniel/.kube/config
2018/09/04 18:34:28 Using current context: k8s.strix.kr-oidc
2018/09/04 18:34:29 Open http://localhost:8000 for authorization
2018/09/04 18:34:29 GET /
...
2018/09/04 18:35:24 Got token for subject=10adf869-a37a-44b2-9589-e1a72afc4ae0
2018/09/04 18:35:24 Updated /Users/daniel/.kube/config

$ kubectl get pods
NAME                      READY     STATUS    RESTARTS   AGE
busybox-57c9c54c8-j9gsc   1/1       Running   0          3d
```

추후에 refresh 토큰이 만료하기 전까지는 다시 인증 할 필요가 없습니다.

#### kube-dashboard keycloak 프록시
kube-dashboard 접속시 HTTP Authentication 헤더를 확인하고 토큰이 없을 시 keycloak으로 인증을 요청하고 토큰을 발급하여 HTTP 헤더에 담아 프록시하면 keycloak의 로그인 계정으로 kube-dashboard의 인증을 대신 할 수 있습니다.

먼저 기존 kube-dashboard ingress를 삭제합니다.
```
$ kubectl delete ingress -n kube-system kube-dashboard
```

그리고 nginx-ingress controller의 프록시 버퍼 사이즈를 키워줍니다. 기본 설정으로는 OIDC의 클레임을 모두 프록시하지 못해서 정상적으로 작동하지 못합니다.
```
$ kubectl patch configmap -n default nginx-ingress-controller -p '{"data": {"proxy-buffer-size": "64k"}}'
```

[프록시 서비스](https://github.com/gambol99/keycloak-proxy)를 띄웁니다. (ref.**14-kube-dashboard-keycloak-proxy.yaml**)

마지막으로 IAM 콘솔에서 kubernetes 클라이언트의 Valid Redirect URLs에 http://k8s.strix.kr/* 을 추가합니다.

### dokuwiki OIDC 적용
dokuwiki의 oauth 플러그인을 설치하고 dokuwiki 클라이언트와 연동합니다.

### kubeapps에 service account 생성 및 접근 제어 적용
kubeapps의 접근 제어에 OIDC는 아직 지원되지 않습니다. (ref. [Github 이슈](https://github.com/kubeapps/kubeapps/issues/385))

OIDC 지원으로 통합 IAM 서비스의 유저 수준의 접근 제어가 가능해지기 전까지, [service account를 통해](https://github.com/kubeapps/kubeapps/blob/master/docs/user/access-control.md) developer와 operator를 구분하는 수준의 접근 제어를 적용합니다.

사전에 정의된 Role 및 ClusterRole들을 생성합니다.
```
# kubeapps-applications-read
$ kubectl -n kubeapps apply -f https://raw.githubusercontent.com/kubeapps/kubeapps/master/docs/user/manifests/kubeapps-applications-read.yaml

# kubeapps-applications-write
# application 설치 권한은 따로 존재하지 않습니다. edit ClusterRole과 kubeapps-repositories-read Role을 바인딩하면 됩니다.

# kubeapps-repositories-read
$ kubectl -n kubeapps apply -f https://raw.githubusercontent.com/kubeapps/kubeapps/master/docs/user/manifests/kubeapps-repositories-read.yaml

# kube-apps-repository-write
$ kubectl -n kubeapps apply -f https://raw.githubusercontent.com/kubeapps/kubeapps/master/docs/user/manifests/kubeapps-repositories-write.yaml
```

developer와 operator용 service account를 생성합니다.
```
$ kubectl create -n kubeapps sa kubeapps-developer
$ kubectl create -n kubeapps sa kubeapps-operator
```

developer service account에 default, dev, prod 네임스페이스의 kubeapps-application-read 권한 및 kubeapps-repositories-read 권한을 추가합니다.
```
$ kubectl create -n default rolebinding kubeapps-developer-bind-kubeapps-applications-read \
  --clusterrole=kubeapps-applications-read \
  --serviceaccount kubeapps:kubeapps-developer

$ kubectl create -n dev rolebinding kubeapps-developer-bind-kubeapps-applications-read \
  --clusterrole=kubeapps-applications-read \
  --serviceaccount kubeapps:kubeapps-developer

$ kubectl create -n prod rolebinding kubeapps-developer-bind-kubeapps-applications-read \
  --clusterrole=kubeapps-applications-read \
  --serviceaccount kubeapps:kubeapps-developer

$ kubectl create -n kubeapps rolebinding kubeapps-developer-bind-kubeapps-repositories-read \
  --role=kubeapps-repositories-read \
  --serviceaccount kubeapps:kubeapps-developer
```

developer service account가 모든 네임스페이스를 확인 할 수 있도록 위에서 생성한 cluster-view ClusterRole을 추가합니다.
```
$ kubectl create clusterrolebinding kubeapps-developer-bind-cluster-view \
  --clusterrole=cluster-view \
  --serviceaccount kubeapps:kubeapps-developer
```

developer service account가 dev 네임스페이스에 소프트웨어를 설치 할 수 있도록 edit ClusterRole 권한을 추가합니다.
```
$ kubectl create -n dev rolebinding kubeapps-developer-bind-edit \
  --clusterrole=edit \
  --serviceaccount kubeapps:kubeapps-developer
```

operator service account에게는 cluster-admin ClusterRole을 바인딩합니다.
```
$ kubectl create clusterrolebinding kubeapps-operator-bind-cluster-admin \
  --clusterrole=cluster-admin \
  --serviceaccount kubeapps:kubeapps-operator
```

아래 명령어로 각 service account의 토큰을 확인 할 수 있습니다.
```
$ kubectl get -n kubeapps secret $(kubectl get -n kubeapps sa kubeapps-developer -o jsonpath='{.secrets[].name}') -o jsonpath='{.data.token}' | base64 --decode; echo

$ kubectl get -n kubeapps secret $(kubectl get -n kubeapps sa kubeapps-operator -o jsonpath='{.secrets[].name}') -o jsonpath='{.data.token}' | base64 --decode; echo
```

## 9. 개발 및 배포 프로세스

### Docker 레지스트리 구성

#### keycloak 클라이언트 추가
keycloak은 docker-v2 인증 프로토콜을 지원하지만 기본적으로 비활성화되어있습니다. kubeapps에 접속하여 keycloak helm 차트의 extraArgs에 "-Dkeycloak.profile.feature.docker=enabled"를 추가하여 재배포합니다. 

keycloak이 다시 실행하면 관리자 콘솔에 접속하여 docker-registry 클라이언트를 master realm에 등록합니다.
AKIAJ6MSY3YE74GKBHVA
F8sZ9H7Upz4LZXWW3D9Wa3exFo89gYCxtaH7nS7B

#### AWS S3 버킷 생성
AWS S3에 새로운 버킷 registry.k8s.strix.kr를 생성하고, S3의 Full Access 권한을 갖는 AWS IAM 유저 registry.k8s.strix.kr를 추가합니다.

#### Docker 레지스트리 배포
다음으로 kubeapps에 접속하여 docker-registry를 default 네임스페이스 배포합니다.
- storage 옵션을 s3로 설정하고 미리 생성한 버킷과 IAM 액세스 토큰 정보를 기입합니다.
- ingress 생성을 활성화하고 registry.k8s.strix.kr 도메인을 할당합니다. 이때 ingress annotation에 `nginx.ingress.kubernetes.io/proxy-body-size: "500m"`을 추가합니다.
- configData 옵션에 keycloak에서 생성한 docker-registry 클라이언트의 installation 항목에서 확인 할 수 있는 [Docker 레지스트리 인증 설정](https://docs.docker.com/registry/configuration) 항목을 복사해 기입합니다. 이때 추가로 rootcertbundle 항목을 기입합니다. 해당 항목은 root CA(keycloak)의 인증서 경로를 가리킵니다. 해당 helm 차트에서 바로 Secret을 마운트 할 수 있는 방법이 없어 우선 root cert의 경로를 임의로 지정합니다.

```
  auth:
    token:
      realm: https://iam.k8s.strix.kr/realms/master/protocol/docker-v2/auth
      service: docker-registry
      issuer: https://iam.k8s.strix.kr/realms/master
      rootcertbundle: /root/ca.pem
```

배포후 `/root/ca.pem` 파일 찾을 수 없어서 오류가 납니다. CA 인증서는 keycloak IAM 콘솔에서 realm 설정 > Keys 탭에서 찾을 수 있습니다. 배포시 생성된 **docker-registry-secret** Secret를 재활용해서 **ca.pem**이라는 키를 추가해줍니다. 이후 해당 Secret을 볼륨에 마운트해서 임의 지정한 경로를 맞춰줍니다.

```
$ echo -n "
-----BEGIN CERTIFICATE-----
<keycloak의 CA 인증서>
-----END CERTIFICATE-----" | base64

$ kubectl patch secret/docker-registry-secret -n default --type json -p '[
  {
    "op": "add",
    "path": "/data/ca.pem",
    "value": "<위에 출력한 base64 인코딩된 ca.pem>"
  }
]'

$ kubectl patch deployment/docker-registry -n default --type json -p '[
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/1",
    "value": {
      "name": "docker-registry-secret",
      "secret": {
        "secretName": "docker-registry-secret"
      }
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/1",
    "value": {
      "name": "docker-registry-secret",
      "mountPath": "/root"
    }
  }
]'
```

이제 https://registry.k8s.strix.kr 로 접속 할 수 있습니다. 만약 503 에러가 발생한다면 [버킷이 비어있는 경우의 버그](https://github.com/docker/distribution/issues/2292#issuecomment-378521123) 일 수 있습니다. 웹 브라우저로의 접속에 문제가 없다면 docker cli로 IAM 연동을 확인합니다.
```
$ docker login https://registry.k8s.strix.kr
Username: donguk.kim@strix.co.kr
Password:
Login Succeeded
```

#### k8s에 이미지 풀링용 Secret 구성
IAM에 가상의 유저 registry.k8s.strix.kr를 등록해주고 해당유저의 정보를 기반으로 Secret 리소스를 생성합니다. 이 Secret **local**은 이후 배포 할때마다 사용됩니다.

```
$ kubectl -n default create secret docker-registry local-docker-registry-secret --docker-server=registry.k8s.strix.kr --docker-username=registry.k8s.strix.kr --docker-password=password

$ kubectl -n dev create secret docker-registry local-docker-registry-secret --docker-server=registry.k8s.strix.kr --docker-username=registry.k8s.strix.kr --docker-password=password

$ kubectl -n prod create secret docker-registry local-docker-registry-secret --docker-server=registry.k8s.strix.kr --docker-username=registry.k8s.strix.kr --docker-password=password
```

매번 Deployment 스펙에 imagePullSecrets 항목을 적어 줄 수도 있지만 각 네임스페이스의 **default** service account에 imagePullSecrets을 [지정](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account)해두면 번거로움이 없어집니다.

```
$ kubectl patch sa default -n default -p '{"imagePullSecrets": [{"name": "local-docker-registry-secret"}]}'
$ kubectl patch sa default -n prod -p '{"imagePullSecrets": [{"name": "local-docker-registry-secret"}]}'
$ kubectl patch sa default -n dev -p '{"imagePullSecrets": [{"name": "local-docker-registry-secret"}]}'
```

#### CI 플랫폼 Jenkins 구축

[Jenkins](https://github.com/jenkinsci/kubernetes-plugin)는 빌드 및 배포 파이프라인을 구성 할 수 있는 오픈소스 CI/CD 소프트웨어입니다. (ref. **15-jenkins-ingress.yaml**)
    - jenkins 서비스를 연결하는 Ingress를 생성합니다. 이제 https://deploy.k8s.strix.kr 로 접속 할 수 있습니다.