# K8S Operation

The granted **system:developer**s and **system:manager**s can use [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [helm](https://docs.helm.sh/helm/) to manage and to view the resources of granted namespaces.

### Setup kubectl and authenticate
(for MacOS)

Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) which is used for manupulate all type of resources in the cluster. And give us many operation feafures like exec into container, cp files between container and localhost, see the container logs, watch the service's status, find specific resource, container port-forwarding, cluster dashboard proxy etc.

```
brew update && brew install kubernetes-cli
```

And manually install [kubelogin](https://github.com/int128/kubelogin) which helps kubectl to be authenticated via STRIX IAM service (https://iam.strix.co.kr).

Now, create the kubeconfig file which is normally stored at `~/.kube/config` and store the credentials and cluster information.

```
echo "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMwekNDQWJ1Z0F3SUJBZ0lNRlU3OFJhc0FRR1dEcUhXcU1BMEdDU3FHU0liM0RRRUJDd1VBTUJVeEV6QVIKQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13SGhjTk1UZ3dPREkyTURjek9ERTVXaGNOTWpnd09ESTFNRGN6T0RFNQpXakFWTVJNd0VRWURWUVFERXdwcmRXSmxjbTVsZEdWek1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBCk1JSUJDZ0tDQVFFQXdWN0N3UkQ3ZkRzOGxkU1huQ3dQS0huYW1Ddk56ekZxVGtOdmVQOUZBanl1VitPSC9nQk0KcDRhKzQ2UXhOWGlhZzRiWEMwanZjSmVRMGVLVkNFZ0FOUkcvYkF6a1dVU2JLcFhSeE1HbnBSbThqaVA4NHRBbApuUFRxandBdjlUVXEvbnlTaVUzMWFjUUlzb2tManJNc1c2ZlJVQTRISnlJbG8zMHFFM3djM1o2MFFlek5nODV4Cm52eUtxc3drWGYzcnF0OW5XMzlGNmgvRnpMVmpqUmxKVUNyVlpPbFQ1cXp5bTJqRVJpK3NMaldLYlpqdTdXcnEKUGFyd0NjQVl5TFdXME12QStlQi84Z1YxVTNyTkV5TDJRL0ljZzI0a3kyaEpCRGpyT21MVTJvRWtTM3FTTEo4TAplbHorRklIVnNvRXdxRDhwSVNoQ1JDRTRkSElOcVl0MXBRSURBUUFCb3lNd0lUQU9CZ05WSFE4QkFmOEVCQU1DCkFRWXdEd1lEVlIwVEFRSC9CQVV3QXdFQi96QU5CZ2txaGtpRzl3MEJBUXNGQUFPQ0FRRUFVMkxUaVBRTUxCSlMKTVZ2Yzh6MUxaK0Npb3FnaHJVWUUvNm9wang1SVU2YmVyZG5meHgzUVkzWms3QXlnamVGVU5NMGJURHpQbUxTRwpDRGgvaHdkaDFuRVYyVUtid2tYRFpPVjhDWUFJYUcrZ1dmMHZrdmxCYjEvVFRWR21KMC9HUThjcDBGcU9HdVptCkE3ejVnREdVM0xwTU14Y1JISzdwRDJxaENZT3BkOGN0YVI0a0h6Z1J3cjRYU2pBaExtMUhtQ2tqZjdlbVZWdUcKQk0zV1hFb0dGVkxSWURYYzFoZVNvWmhlU0VQdGVwd2kwTi9ZSzdiNVorb3FEV09lT0h5UktuZ1pVUTNWQmo3OQptbGo5R3FmTklkaU9NTVEwVXVyZjRsbHpDbGhwWlVWQlJDSTNPMmt1cjRERFlITWJPMDczSElwdmtEUUlOdUpGCnE0bUdlUjJ2YWc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==" | base64 --decode > ca.key && kubectl config set-cluster "k8s.strix.kr" --server=https://api.k8s.strix.kr --embed-certs=true --certificate-authority=./ca.key; rm ca.key
kubectl config set-credentials "k8s.strix.kr-oidc" --auth-provider oidc --auth-provider-arg idp-issuer-url=https://iam.strix.kr/auth/realms/default --auth-provider-arg client-id=kubernetes --auth-provider-arg client-secret=343deb41-052a-46fc-b4c9-27ee898f0717
kubectl config set-context k8s.strix.kr-oidc --namespace dev --cluster=k8s.strix.kr --user=k8s.strix.kr-oidc && kubectl config use-context k8s.strix.kr-oidc
kubelogin
```

Test can you view default pods in the namespace.
```
kubectl get pods -n default
```

### To manipulate cluster resources
...


## Local development and debugging between remote cluster

Install [telepresence](https://www.telepresence.io/reference/install). (macOS)
```
brew cask install osxfuse && brew install socat datawire/blackbird/telepresence
```

kube-prompt or kube-shell will help


### See more
See https://kubernetes.io/docs/reference/kubectl/cheatsheet/

### If you have another cluster kubeconfig already
```
kubectl config get-contexts/use-context
```
...


stuck in trouble
https://console.aws.amazon.com/route53/home?region=ap-northeast-2#resource-record-sets:ZSOAG5RQYWIDL
https://ap-northeast-2.console.aws.amazon.com/ec2/v2/home?region=ap-northeast-2#Instances:sort=desc:launchTime
https://github.com/strix-kr/k8s
https://github.com/rmmh/k8s-kops/blob/master/docs/networking.md
https://github.com/projectcalico/cni-plugin/issues/434

kops rolling-update cluster --force --yes   --fail-on-validate-error="false"   --node-interval 1m --master-interval 1m