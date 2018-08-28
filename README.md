# K8S Management

A brief description for the k8s cluster management.

## 1. For Cluster Administrator

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
kops export kubecfg --name k8s.strix.kr --state s3://k8s.strix.kr
```

### To update cluster spec
```
kops edit cluster
kops update cluster --yes
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

### More
See https://github.com/kubernetes/kops/blob/master/docs/cli/kops.md

## 2. Default Services

---
Endpoint|Protocol|Usage
---

## 3. For Developer and Manager

The granted **developer**s and **manager**s can use [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [helm](https://docs.helm.sh/helm/) to manage and to view the resources of granted namespaces.

