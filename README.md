# 인프라 관리 매뉴얼
## 1. 인프라 구성 이력
### A. Kubernetes 클러스터 on AWS

#### 설치 기록
[kops](https://github.com/kubernetes/kops)는 kubeadm를 활용한 k8s 클러스터 구성 및 관리 도구입니다. [AWS용 가이드](https://github.com/kubernetes/kops/blob/master/docs/aws.md)를 참고하여 사전 조건을 완료합니다.
```
- kubectl, kops CLI 설치
- kops용 액세스키 생성 및 권한 부여 (AWS IAM)
- 버저닝이 활성화된 kops용 클러스터 상태 저장소 생성 (AWS S3)
- k8s API 도메인용 DNS 호스트존 구성 (AWS Route53)
```

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
kops create cluster \
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

#### 문제 해결
1. [롤링 업데이트](https://github.com/kubernetes/kops/blob/master/docs/cli/kops_rolling-update.md) 도중 예기치 못한 이유로 노드 방출에 실패하여 클러스터가 마비되는 경우.
```
kops validate cluster
kubectl describe node
kubectl cluster-info dump
watch kubectl get deploy --all-namespaces
watch kubectl get pod --all-namespaces
```
위 명령어 등으로 문제점을 찾습니다. 만약 특정 pod을 제거하지못하는 경우 아래 명령어로 pod을 제거하여 수동으로 노드를 방출하도록 도울 수 있습니다.
```
kubectl delete pod <셀렉터> --grace-period=0 --force
```
문제가 되는 pod이 많은 경우 아래 명령어로 다수의 노드를 제거하는 명령을 출력 할 수 있습니다.
```
kubectl get pods --all-namespaces -o wide | grep <키워드> | awk '{print "kubectl delete pod", $2, "-n", $1, "--force --grace-period=0"}'
```
kubectl이나 kops가 마비되는 경우, DNS를 확인하고 수동으로 api.k8s.strix.kr 을 API 엔드포인트에 연결하세요. 긴급한 경우 빠른 반영을 위해서 로컬 DNS 서버의 캐시를 flush 할 수도 있습니다. [ref. 구글 DNS 서버](https://developers.google.com/speed/public-dns/cache)

외부 k8s 대시보드 엔드포인트가 마비되는 경우엔 프록시를 열고 로컬에서 관리자의 service account 토큰으로 접속합니다. 이 때 토큰에 대해서는 아래에서 설명합니다.
```
kubectl proxy
open http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

### B. DNS 서비스 제공자
클러스터 생성 후 kops를 이용해서 클러스터의 DNS 서비스 제공자를 [CoreDNS](https://coredns.io/plugins/kubernetes/)로 [변경](https://github.com/kubernetes/kops/blob/master/docs/cluster_spec.md#kubedns)하고 클러스터를 [업데이트](https://github.com/kubernetes/kops/blob/master/docs/changing_configuration.md)합니다. 이 때 롤링 업데이트는 필요하지 않습니다.

클러스터 생성시 기본 DNS 서비스 제공자는 **kube-dns**로 지정되어 있습니다. 클러스터에 **coredns**가 배포된 이후 kube-dns를 제거하고 dns-auto-scaler 애드온의 배포 커맨드를 수정합니다.

```
kubectl delete --namespace=kube-system deployment kube-dns
kubectl patch deployment/kube-dns-autoscaler -n kube-system --type='json' -p '[
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

coredns는 kube-dns에 비해 DNS 서비스를 구성 할 수 있는 다양한 기능을 제공하며, 그 구성을 k8s API로 [손쉽게 편집](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#coredns-configmap-options) 할 수 있습니다.
  - [DNS 포워딩](https://coredns.io/plugins/proxy/):
    타 네임서버로 쿼리 요청을 위임할 수 있습니다.
  - [DNS 페더레이션](https://coredns.io/plugins/federation/):
    클러스터 레벨의 결합이 필요한 경우 클러스터간의 서비스 탐지를 결합 할 수 있습니다.
  - [로컬 존 파일 적용 (/etc/hosts)](https://coredns.io/plugins/hosts/):
  - [기타 플러그인들](https://coredns.io/plugins/)

### C. 네트워크 정책
[kube-router](https://github.com/cloudnativelabs/kube-router)는 k8s 클러스터에 CNI 레이어를 담당하는 애드온으로 설치되었습니다. 현 시점에서 k8s 클러스터의 기본 구성은 k8s network policy API의 스펙을 실제로 구현하지 않고 있습니다. 이 때문에 클러스터에 namespace/pod/ip/port 등의 리소스를 기준으로 inbound/outbound 방화벽 정책을 설정하려면 이 같은 별도의 애드온이 필요합니다. [ref.](https://github.com/kubernetes/kops/blob/master/docs/networking.md)

네트워크 정책 리소스 업데이트시 kube-router가 [networking.k8s.io/NetworkPolicy API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#networkpolicy-v1-networking-k8s-io) 스펙을 지원하는 점에 유의해야합니다. 또한 현 시점(18.09.01)에 `namespaceSelector.matchExpressions`가 구현되지 않았기에 `namespaceSelector.matchLabels`를 이용합니다.

#### 네임스페이스간 방화벽
아래의 다이어그램은 네임스페이스간에 허용되는 트래픽을 보여줍니다.
```
prod (label: env=prod) <-> default/kube-system/kube-public (label: env=common) <-> dev (label: env=dev)
```

#### 네트워크 정책 리소스 업데이트시의 방화벽 테스트 도구
prod, default, dev의 각 네임스페이스에 **busybox**라는 이름으로 service/deployment가 등록했습니다.

배포된 pod들은 네트워크 정책 디버깅 용도로 생성되었습니다. **4-busyboxes-test** 툴을 이용해서 손쉽게 트래픽을 점검 할 수 있습니다.

현재 방화벽 정책에 대한 올바른 테스트 결과입니다.
```
dehypnosis-mac:k8s dehypnosis$ ./4-busyboxes-test default dev
 * Send HTTP request [default -> dev]
 * ip-172-20-36-62.ap-northeast-2.compute.internal/default/busybox-57c9c54c8-gxrrl -> busybox.dev
<xmp>
Hello World


                                       ##         .
                                 ## ## ##        ==
                              ## ## ## ## ##    ===
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


                                       ##         .
                                 ## ## ##        ==
                              ## ## ## ## ##    ===
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


                                       ##         .
                                 ## ## ##        ==
                              ## ## ## ## ##    ===
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


                                       ##         .
                                 ## ## ##        ==
                              ## ## ## ## ##    ===
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
KR_POD=$(basename $(kubectl -n kube-system get pods -l k8s-app=kube-router --output name|head -n1))
kubectl -n kube-system exec -it ${KR_POD} bash
```

### D. 패키지 설치 도구

#### helm 및 tiler/kubeapps 설치
kubectl와 yaml 파일들로 소프트웨어 패키지를 관리 할 수 있지만, 복잡한 소프트웨어를 클러스터에 설치하고 관리하는 일을 더 쉽게하기 위해 [helm](https://github.com/helm/helm)  패키지 매니저를 사용합니다.

로컬과 클러스터에 각각 client(helm), server(tiler)를 초기화합니다. 추후 다른 머신에서는 로컬의 helm만 초기화하게 됩니다.
```
helm init
```

다음으로 tiler에게 service account를 생성해주고 클러스터에 전체에 관한 권한을 위임합니다. 또한 helm chart(패키지) 관리를 돕는 웹 서비스 [kubeapps](https://github.com/kubeapps/kubeapps)를 설치합니다. (ref. **5-install-tiler-and-kube-apps**)

#### kubeapps를 프록시하여 접속
지금까지 관리자가 kubectl을 통해서 k8s API 서버에 PKI로 인증을 했지만, 여러 k8s 서비스에서 관리자가 웹 브라우저에서 로그인 할 때는 토큰 방식의 인증을 이용합니다.
여기서 생성한 service account의 토큰은 이후에 k8s 대시보드 같은 k8s API의 cluster-admin 권한이 필요한 모든 서비스에 접근 할 때도 이용 할 수 있습니다. (ref. **6-admin-service-token**)

```
kubectl get secret $(kubectl get sa admin -o jsonpath='{.secrets[].name}') -o jsonpath='{.data.token}' | base64 --decode; echo
```
위처럼 관리자 service account 토큰을 확인 할 수 있습니다.

```
open http://127.0.0.1:8080
kubectl port-forward --namespace kubeapps svc/kubeapps 8080:80
```
당장 웹 서버가 없기에 kubectl port-forward를 통해 로컬에서 접속합니다.

#### 설치된 패키지
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
  대시보드는 k8s 클러스터에 등록된 모든 리소스에 대한 명세를 확인하고 수정 할 수 있는 웹 UI를 제공합니다. 또한 클러스터 node 및 각 pod들이 소비하는 CPU/RAM에 대한 메트릭을 제공합니다. 또한 웹 상에서 각 pod에서 수집된 로그를 확인하고 shell을 구동 할 수 있습니다.
    - kube-dashboard 서비스를 연결 하는 Ingress를 생성합니다. 마찬가지로 관리자 service account 토큰으로 접속합니다. 이제 https://k8s.strix.kr 로 접속 할 수 있습니다.
- [kube-system/kube-backup](https://github.com/pieterlange/kube-backup):
  주기적으로 모든 리소스의 스펙을 특정 git 저장소에 백업하는 배치 프로세스를 생성합니다.
    - Secret 리소스는 [git-crypt](https://github.com/AGWA/git-crypt)로 대칭키 암호화되어 커밋됩니다.
    - helm 차트가 제공되지 않으므로 수동으로 설치되었습니다.
- [kube-system/kube-db](https://kubedb.com):
  데이터베이스를 써드파티 리소스로 정의하여 k8s 클러스터 내에 손쉽게 생성 할 수 있습니다. 자동화된 백업, 복원 기능 및 **kubedb**라는 전용 CLI를 제공합니다.
    - kubedb의 인터페이스는 kubectl과 거의 동일하며 kubectl로도 kubedb가 제공하는 리소스를 모두 조작 할 수 있습니다. 다만 kubedb는 데이터베이스에 한정된 리소스만을 조작하며, 추가적으로 위험한 명령에 대한 검증 및 디버깅 관련 인터페이스를 제공합니다.
- [default/dokuwiki](https://www.dokuwiki.org/ko:dokuwiki):
  오픈소스 위키 소프트웨어입니다.
    - dokuwiki 서비스를 연결하는 Ingress를 생성합니다. 이제 https://wiki.strix.kr 로 접속 할 수 있습니다.
- [default/keycloak](https://www.keycloak.org):
  오픈소스 통합 IAM 소프트웨어입니다. 이 통합 IAM 서비스에 RBAC 체계로 지금까지 등록한 내부 서비스의 접근을 제어하게됩니다. 또한 외부 서비스의 인증 및 접근 제어 역시 통합 IAM에 등록되어 관리됩니다.
    - 본 서비스는 클러스터 내의 대부분의 계정 관리를 담당하기 때문에 데이터베이스의 주기적인 백업이 필요합니다.