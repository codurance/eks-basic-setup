# Steps to Create an EKS Cluster

The setup from the `eksctl-setup` folder is slightly simpler, but a lot less detailed. It was used as a reference to produce the below commands and scripts in conjunction with this blog post: https://aws.amazon.com/blogs/containers/using-alb-ingress-controller-with-amazon-eks-on-fargate/ , and the AWS documentation. Many of the AWS docs have been referenced below.

- [x] Create a VPC, an EKS Cluster, roles for EKS, and an EKS Fargate Profile.

```bash
STACK_NAME=roche-eks-environment
CLUSTER_NAME=roche-eks-cluster
REGION=eu-central-1
PROFILE=codurance-user-playground

aws --region $REGION --profile $PROFILE \
  cloudformation create-stack \
  --stack-name $STACK_NAME \
  --template-body file://eks-environment.yaml \
  --tags Key=name,Value=roche \
  --capabilities CAPABILITY_NAMED_IAM

# To update the Cloud Formation stack with new changes.
aws --region $REGION --profile $PROFILE \
  cloudformation update-stack \
  --stack-name $STACK_NAME \
  --template-body file://eks-environment.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# To delete the Cloud Formation stack and all resources.
aws --region $REGION --profile $PROFILE \
  cloudformation delete-stack \
  --stack-name $STACK_NAME
```

- [x] Update the kube config.

```bash
# If an error appears when running the below command then,
# rm -rf ~/.kube/cache ~/.kube/config.
aws --region $REGION --profile $PROFILE \
  eks update-kubeconfig --name $CLUSTER_NAME

# Test the connection with the cluster.
kubectl get svc
```

- [x] Patch the coredns pods to allow them to work on Fargate.

```bash
kubectl patch deployment coredns \
  -n kube-system \
  --type json \
  -p='[{"op": "remove", "path": "/spec/template/metadata/annotations/eks.amazonaws.com~1compute-type"}]'
```

- [x] Set up an OIDC Provider using the AWS Management Console following the steps from this url: https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html .

```bash
# Get the Provider URL with the following command.
aws --region $REGION --profile $PROFILE \
  eks describe-cluster \
  --name $CLUSTER_NAME \
  --query 'cluster.identity.oidc.issuer' \
  --output text
```

- [x] Create an ingress IAM policy.

```bash
curl -o alb-ingress-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/iam-policy.json

aws --region $REGION --profile $PROFILE \
  iam create-policy \
  --policy-name RocheALBIngressControllerIAMPolicy \
  --policy-document file://alb-ingress-iam-policy.json
```

- [x] Create a cluster role, a role binding, and a k8s service account.

```bash
VPC_ID=$(aws --region "$REGION" --profile "$PROFILE" \
  cloudformation describe-stacks \
  --stack-name "$STACK_NAME" \
  | jq -r '[.Stacks[0].Outputs[] | {key: .OutputKey, value: .OutputValue}] | from_entries' | jq -r '.VpcId')

AWS_ACCOUNT_ID=$(aws --region "$REGION" --profile "$PROFILE" \
  sts get-caller-identity \
  | jq -r '.Account')

# Update alb-ingress-role.json to use the correct OIDC_PROVIDER_ID and create
# role with the  previously created RocheALBIngressControllerIAMPolicy.
# Reference: https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html
aws --region $REGION --profile $PROFILE \
  iam create-role \
  --role-name RocheEksClusterServiceAccountRole \
  --assume-role-policy-document file://alb-ingress-role.json

# Attach the RocheALBIngressControllerIAMPolicy to the RocheEksClusterServiceAccountRole.
aws --region $REGION --profile $PROFILE \
  iam attach-role-policy \
  --role-name RocheEksClusterServiceAccountRole \
  --policy-arn=arn:aws:iam::300563897675:policy/RocheALBIngressControllerIAMPolicy

# Create ClusterRole and ClusterRoleBinding.
# Reference: https://aws.amazon.com/blogs/containers/using-alb-ingress-controller-with-amazon-eks-on-fargate/
kubectl apply -f rbac-role.yaml

# Create service account.
# Edit AWS_ACCOUNT_ID and IAM_ROLE_NAME, i.e., associate the service account with RocheEksClusterServiceAccountRole.
# Reference: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
# Reference: https://docs.aws.amazon.com/eks/latest/userguide/specify-service-account-role.html
kubectl apply -f service-account.yaml -n kube-system

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