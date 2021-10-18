AWS - The YAML way
------------------

![](/images/aws-the-yaml-way.jpg)
## Motivation

1. I want to manage AWS infrastructure with YAML
2. I want to be able to defines rules to govern my cloud resources

## But why?

Why not :)
## How?

- For (1), there's AWS Controllers for Kubernetes (ACK) or maybe [Crossplane](https://crossplane.io/)
- For (2), I'm thinking Kyverno or OpenPolicyAgent.

## Let's do it

### Setup the env

I will skip the part where you setup AWS CLI and an EKS cluster Suppose that is all set and done. If not, follow the brief instructions below to setup a new EKS cluster. The easiest way IMO is to use `eksctl` from Weaveworks.

```sh
aws ec2 create-key-pair --region ap-southeast-1 --key-name my-yaml-eks-key

eksctl create cluster \
    --name my-yaml-eks \
    --region ap-southeast-1 \
    --with-oidc \
    --ssh-access \
    --ssh-public-key my-yaml-eks-key \
    --managed
```

Wait for a bit for the cluster to be provisioned.

There will be 2 CloudFormation stacks being provisioned so it might take awhile. If sth goes wrong (disconnection, etc..), you can check with this command below

```sh
eksctl utils describe-stacks --region=ap-southeast-1 --cluster=my-yaml-eks
```

```sh
2021-10-18 22:35:28 [ℹ]  eksctl version 0.70.0
2021-10-18 22:35:28 [ℹ]  using region ap-southeast-1
2021-10-18 22:35:29 [ℹ]  setting availability zones to [ap-southeast-1a ap-southeast-1b ap-southeast-1c]
2021-10-18 22:35:29 [ℹ]  subnets for ap-southeast-1a - public:192.168.0.0/19 private:192.168.96.0/19
2021-10-18 22:35:29 [ℹ]  subnets for ap-southeast-1b - public:192.168.32.0/19 private:192.168.128.0/19
2021-10-18 22:35:29 [ℹ]  subnets for ap-southeast-1c - public:192.168.64.0/19 private:192.168.160.0/19
2021-10-18 22:35:29 [ℹ]  nodegroup "ng-5c441b7d" will use "" [AmazonLinux2/1.20]
2021-10-18 22:35:29 [ℹ]  using EC2 key pair %!q(*string=<nil>)
2021-10-18 22:35:29 [ℹ]  using Kubernetes version 1.20
2021-10-18 22:35:29 [ℹ]  creating EKS cluster "my-yaml-eks" in "ap-southeast-1" region with managed nodes
2021-10-18 22:35:29 [ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
2021-10-18 22:35:29 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-southeast-1 --cluster=my-yaml-eks'
2021-10-18 22:35:29 [ℹ]  CloudWatch logging will not be enabled for cluster "my-yaml-eks" in "ap-southeast-1"
2021-10-18 22:35:29 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=ap-southeast-1 --cluster=my-yaml-eks'
2021-10-18 22:35:29 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "my-yaml-eks" in "ap-southeast-1"
2021-10-18 22:35:29 [ℹ]
2 sequential tasks: { create cluster control plane "my-yaml-eks",
    2 sequential sub-tasks: {
        4 sequential sub-tasks: {
            wait for control plane to become ready,
            associate IAM OIDC provider,
            2 sequential sub-tasks: {
                create IAM role for serviceaccount "kube-system/aws-node",
                create serviceaccount "kube-system/aws-node",
            },
            restart daemonset "kube-system/aws-node",
        },
        create managed nodegroup "ng-5c441b7d",
    }
}
2021-10-18 22:35:29 [ℹ]  building cluster stack "eksctl-my-yaml-eks-cluster"
2021-10-18 22:35:30 [ℹ]  deploying stack "eksctl-my-yaml-eks-cluster"
```

Once it's done. Make sure the cluster is accessible

![](/images/eks-ready.png)

### Setup Crossplane

At first, I plan to use ACK but then I remember a friend of mine talked about Crossplane the other day so I want to give it a try. Bonus point, it's compatible with multiple cloud providers :)

I'm going to use Helm here so make sure you have it installed. I just blindly follow the official [installation instructions on Crossplane website here](https://crossplane.io/docs/v1.4/getting-started/install-configure.html).

As of this post, `1.4.1` is the latest chart version.

```sh
kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane --version 1.4.1
```

```
NAME: crossplane
LAST DEPLOYED: Mon Oct 18 23:05:25 2021
NAMESPACE: crossplane-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Release: crossplane

Chart Name: crossplane
Chart Description: Crossplane is an open source Kubernetes add-on that enables platform teams to assemble infrastructure from multiple vendors, and expose higher level self-service APIs for application teams to consume.
Chart Version: 1.4.1
Chart Application Version: 1.4.1

Kube Version: v1.20.7-eks-d88609
```

Check the status

```sh
helm list -n crossplane-system
kubectl get all -n crossplane-system
```

![](/images/crossplane-installed.png)

Also, make sure to install the Crossplane CLI.

```sh
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
```

### Setup Crossplane provider for AWS

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: crossplane/provider-aws:alpha
```

After the provider is ready, apply this next. You need to do this in 2 steps because otherwise, `ProviderConfig` kind is unknown.

```yaml
apiVersion: v1
stringData:
  config: |
    [default]
    aws_access_key_id = <your-aws-access-key-id>
    aws_secret_access_key = <your-aws-secret-access-key>
kind: Secret
metadata:
  creationTimestamp: null
  name: aws-credentials
---
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-provider-config
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-credentials
      key: config
```

Check the status of the provider

```sh
kubectl get providers
```

![](/images/provider-aws-ready.png)

### Let's try to create some resources on AWS

Let's do a S3 bucket first. Make sure you do a random bucket name so that it won't conflict with an existing bucket.

```yaml
apiVersion: s3.aws.crossplane.io/v1beta1
kind: Bucket
metadata:
  name: test-bucket-1923981293819238
  namespace: default
spec:
  providerConfigRef:
    name: aws-provider-config
  forProvider:
    locationConstraint: ap-southeast-1
    acl: private
```

![](/images/s3-created.png)

So that's cool. What's next? Let's trying to use OPA or Kyverno to set some rules for our newly created resources.

### Setup Kyverno

I'm gonna go with Kyverno here. No particular reason. I just feel OPA is too main stream :D

Let's install Kyverno with Helm

```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno-crds kyverno/kyverno-crds --namespace kyverno --create-namespace
helm install kyverno kyverno/kyverno --namespace kyverno
```

Now, let's write a simple policy. Say, we don't want to allow creating S3 bucket anywhere else except in `ap-southeast-1` region.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: allow-s3-ap-southeast-1-only
  annotations:
    pod-policies.kyverno.io/autogen-controllers: none
    policies.kyverno.io/title: Allow creating S3 bucket in ap-southeast-1 only
    policies.kyverno.io/subject: Bucket
    policies.kyverno.io/description: >-
      Allow creating S3 bucket in ap-southeast-1 only
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: s3-in-ap-southeast-1-only
    match:
      resources:
        kinds:
        - Bucket
    validate:
      message: "Using any location other than `ap-southeast-1` is not allowed"
      pattern:
        spec:
          forProvider:
            locationConstraint: "ap-southeast-1"
```

Now, let's try to create a S3 bucket in `us-east-1` region. You will be blocked by the admission controller.

```yaml
# bad-s3.yaml
apiVersion: s3.aws.crossplane.io/v1beta1
kind: Bucket
metadata:
  name: test-bucket-091289310923801928309123
  namespace: default
spec:
  providerConfigRef:
    name: aws-provider-config
  forProvider:
    locationConstraint: us-east-1
    acl: private
```

```sh
kubectl create -f bad-s3.yaml
```

![](/images/bad-s3.png)

## Conclusion

I'm not going to tell you should start doing this but it's a feasible way of managing infrastructure at small scale.

Also with the release of AWS Cloud Control API, the resources model is very close with the Kubernetes resources model now. I'm expecting ACK to get much much better.

Have fun hacking AWS :)