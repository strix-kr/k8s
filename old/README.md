# K8S Management

A brief description for the k8s cluster management.

## For Cluster Administrator

### AWS Resources for kops (IAM, DNS, State Storage)

The cluster is deployed on AWS EC2 on ap-northeast-2a region.

The AWS IAM user/group **kops/kops** have been made and have below AWS IAM permissions.

```
AmazonEC2FullAccess
AmazonRoute53FullAccess
AmazonS3FullAccess
IAMFullAccess
AmazonVPCFullAccess
```

And the cluster's internal DNS will use the domain suffixed with **.k8s.strix.kr**. Currently the DNS records of this domain is managed by AWS Route53.

And kops is using AWS S3 bucket **k8s-strix-kr** to store and version control the current state of the cluster specification.

In addition, IAM user **cert-manager** has `AmazonRoute53FullAccess` permission for ACME protocol.

### Setup kops and authenticate
[kops](https://kubernetes.io/docs/tasks/tools/install-kubectl) is a CLI client program to manage and access the resources inside the k8s cluster.

(for MacOS)
```bash
brew update && brew install kubernetes-cli kops
```

Now to access current cluster state, configure the AWS credentials as the user **kops** (Get the credentials from AWS IAM console).

```bash
export AWS_ACCESS_KEY_ID=XXXXX
export AWS_SECRET_ACCESS_KEY=XXXXX
export NAME=k8s.strix.kr
export KOPS_STATE_STORE=s3://k8s-strix-kr
```

And create the kubectl config file from the current cluster state.
```
kops export kubecfg --name $NAME --state $KOPS_STATE_STORE
```

### To update cluster spec
```
kops edit cluster
kops update cluster --yes
```

if the changes require instances to restart,
```
kops rolling-update cluster --yes
```

### To update instance group spec
See current intance groups.
```
kops get $NAME
```

Update the group spec.
```
kops edit ig <group>
kops update cluster --yes
```

if the changes require instances to restart,
```
kops rolling-update cluster --yes
```

### To upgrade k8s
```
kops upgrade cluster --yes
kops update cluster --yes
kops rolling-update cluster --yes
```

### for disaster on rolling update
try to turn off validation feature on rolling-update
https://github.com/kubernetes/kops/blob/master/docs/cli/kops_rolling-update.md

kops validate cluster
when stuck .. draining pod: see describe nodes .. unterminated pod, and delete.
kubectl describe nodes
kubectl cluster-info dump
see aws console ec2 status
ssh into node and master node

### when external dashboard ingress crashed
kubectl proxy
then enter blow, need jwt access token, see kubectl config view, find user made from kops export kubecfg
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

### network policy not applied correctly
see rule update log
kubectl logs -l name=weave-net -c weave-npc -n kube-system

see router update log
kubectl logs -l name=weave-net -c weave -n kube-system

recreate all pods of weave daemonset will help
kubectl delete pod -l name=weave-net -n kube-system

debug with busybox on dev, prod, default
kubectl get pods --all-namespaces -l app=busybox -o wide

test with scaled pods
kubectl scale --replicas=5 deploy busybox -n dev

like this
kubectl exec busybox-57c9c54c8-lmf7j -n prod -- wget busybox.default

or.. let all pot od busybox.prod wget busybox.default
export WGET_FROM=prod
export WGET_TO=default
kubectl get pods -n $WGET_FROM -l app=busybox -o wide | awk -v FROM="$WGET_FROM" -v TO="$WGET_TO" '{ if (NR>1) print "echo \x27 * " FROM, ",", $1, "," $7 ")", "->", FROM "\x27;", "kubectl exec -n" FROM, $1, "-- wget -O- busybox." TO }' | bash

### restart whole k8s process
enter master node via ssh and as root
```
systemctl stop kubelet
systemctl stop docker
iptables --flush
iptables -tnat --flush
systemctl start kubelet
systemctl start docker
```

### kill stucked pod
kubectl delete pod -l k8s-app=kube-dns -n kube-system --grace-period=0 --force

### immediately evict all pods node
// kill all pods from specific node immediately
export NODENAME="abcdefg"
kubectl get pods --all-namespaces -o wide | grep $NODE_NAME | awk '{print "kubectl delete pod", $2, "-n", $1, "--force --grace-period=0"}' | bash

### Installed  CNI plugin
calico via kops: https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/

### See more
See https://github.com/kubernetes/kops/blob/master/docs/cli/kops.md

Check network policy
Test nginx redirection ingress
Change kube-dashboard-proxy as common service if need

Kube-apps https://github.com/kubeapps/kubeapps/blob/master/docs/user/access-control.md
kube-less
kube-db
kube-backup
kafka https://github.com/krallistic/kafka-operator
keyclock create client policy https://www.keycloak.org/docs/3.3/server_admin/topics/admin-console-permissions/fine-grain.html

circleCI or codefresh

event-service
event-adapter-go
file-upload-service
http-demo-service
slack-notifier-service
sms-notifier-service
k8s-service-template

With document
wiki markdown https://github.com/bharley/mw-markdown/blob/master/README.md