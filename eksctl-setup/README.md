# Steps to Create an EKS Cluster with EKSCTL

Based on: https://aws.amazon.com/blogs/containers/using-alb-ingress-controller-with-amazon-eks-on-fargate/

Note: You will have issues using eksctl using the `codurance-user-playground` SSO setup. You could go through the steps below using a dedicated user. You could restrict the user to only have programatic access which is a common approach for a pipeline.

- [x] Create a cluster with Fargate compute.

```bash
AWS_REGION=eu-central-1
CLUSTER_NAME=eks-fargate-alb-demo

eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --fargate
```

- [x] Update the kube config.

```bash
# If an error appears, rm -rf ~/.kube/cache ~/.kube/config.
aws eks update-kubeconfig \
  --name $CLUSTER_NAME

# Test the connection with the cluster.
kubectl get svc
```

- [x] Set up an OIDC Provider.

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --approve
```

- [x] Create the ingress IAM policy.

```bash
curl -o alb-ingress-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/iam-policy.json
aws iam create-policy \
  --policy-name ALBIngressControllerIAMPolicy \
  --policy-document file://alb-ingress-iam-policy.json
```

- [x] Create a cluster role, a role binding, and a k8s service account.

```bash
STACK_NAME=eksctl-$CLUSTER_NAME-cluster
VPC_ID=$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" | jq -r '[.Stacks[0].Outputs[] | {key: .OutputKey, value: .OutputValue}] | from_entries' | jq -r '.VPC')
AWS_ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')

kubectl apply -f rbac-role.yaml

eksctl create iamserviceaccount \
  --name alb-ingress-controller \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/ALBIngressControllerIAMPolicy \
  --approve

# Check if the account has been created.
kubectl get serviceaccounts/alb-ingress-controller -n kube-system -o yaml
```

- [x] Deploy an ALB (Application Load Balancer) Ingress Controller.

```bash
# Edit --cluster-name, --aws-vpc-id, and --aws-region in the yaml file.
kubectl apply -f alb-ingress-controller.yaml
```

- [x] Deploy a sample app.

```bash
# Create an nginx deployment and service.
kubectl apply -f nginx.yaml

# Check if the deployment is up and running.
kubectl get all -n default

# Create an ingress resource.
kubectl apply -f nginx-ingress.yaml
```

- [x] Get the ALB URL.

```bash
# The URL will appear under ADDRESS in a few minutes.
kubectl get ingress nginx-ingress
```

- [x] Perform an ALB healthcheck.

```bash
# The last command should print 'healthy' 3 times. It might take a few retries in the timespan of a few minutes.
LOADBALANCER_PREFIX=$(kubectl get ingress nginx-ingress -o json | jq -r '.status.loadBalancer.ingress[0].hostname' | cut -d- -f1)
TARGETGROUP_ARN=$(aws elbv2 describe-target-groups | jq -r '.TargetGroups[].TargetGroupArn' | grep $LOADBALANCER_PREFIX)
aws elbv2 describe-target-health --target-group-arn $TARGETGROUP_ARN | jq -r '.TargetHealthDescriptions[].TargetHealth.State'

# Check if all pods are running.
kubectl get pods -o wide
```
