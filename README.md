# Steps to Create an EKS Cluster With Fargate Compute

The setup from the `eksctl-setup` folder is slightly simpler, but a lot less detailed. It was used as a reference to produce the below commands and scripts in conjunction with this blog post: https://aws.amazon.com/blogs/containers/using-alb-ingress-controller-with-amazon-eks-on-fargate/ , and the AWS documentation: https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html. Many of the AWS docs have been referenced below.

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

ALB_INGRESS_POLICY_NAME=RocheALBIngressControllerIAMPolicy

aws --region $REGION --profile $PROFILE \
  iam create-policy \
  --policy-name $ALB_INGRESS_POLICY_NAME \
  --policy-document file://alb-ingress-iam-policy.json
```

- [x] Create a cluster role, a role binding, and a k8s service account.

```bash
AWS_ACCOUNT_ID=$(aws --region "$REGION" --profile "$PROFILE" \
  sts get-caller-identity \
  | jq -r '.Account')

# Update alb-ingress-role.json to use the correct OIDC_PROVIDER_ID and create
# role with the  previously created RocheALBIngressControllerIAMPolicy.
# Reference: https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html
PROVIDER_URL_ID=$(aws --region $REGION --profile $PROFILE \
  eks describe-cluster \
  --name $CLUSTER_NAME \
  --query 'cluster.identity.oidc.issuer' \
  --output text | sed -e 's/.*id\///')

cat > alb-ingress-role.json <<-EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$AWS_ACCOUNT_ID:oidc-provider/oidc.eks.$REGION.amazonaws.com/id/$(echo $PROVIDER_URL_ID)"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.$REGION.amazonaws.com/id/$(echo $PROVIDER_URL_ID):sub": "system:serviceaccount:kube-system:alb-ingress-controller",
          "oidc.eks.$REGION.amazonaws.com/id/$(echo $PROVIDER_URL_ID):aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

SERVICE_ACCOUNT_ROLE_NAME=RocheEksClusterServiceAccountRole

aws --region $REGION --profile $PROFILE \
  iam create-role \
  --role-name $SERVICE_ACCOUNT_ROLE_NAME \
  --assume-role-policy-document file://alb-ingress-role.json

# Attach the RocheALBIngressControllerIAMPolicy to the RocheEksClusterServiceAccountRole.
aws --region $REGION --profile $PROFILE \
  iam attach-role-policy \
  --role-name $SERVICE_ACCOUNT_ROLE_NAME \
  --policy-arn=arn:aws:iam::$AWS_ACCOUNT_ID:policy/$ALB_INGRESS_POLICY_NAME

# Create ClusterRole and ClusterRoleBinding.
# Reference: https://aws.amazon.com/blogs/containers/using-alb-ingress-controller-with-amazon-eks-on-fargate/
kubectl apply -f rbac-role.yaml

# Create a service account and associate it with the RocheEksClusterServiceAccountRole.
# Reference: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
# Reference: https://docs.aws.amazon.com/eks/latest/userguide/specify-service-account-role.html
SERVICE_ACCOUNT_ROLE_ARN=arn:aws:iam::$AWS_ACCOUNT_ID:role/$SERVICE_ACCOUNT_ROLE_NAME

cat > service-account.yaml <<-EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alb-ingress-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: $SERVICE_ACCOUNT_ROLE_ARN
EOF

kubectl apply -f service-account.yaml -n kube-system

# Check if the account has been created.
kubectl get serviceaccounts/alb-ingress-controller -n kube-system -o yaml
```

- [x] Deploy an ALB (Application Load Balancer) Ingress Controller.

```bash
VPC_ID=$(aws --region "$REGION" --profile "$PROFILE" \
  cloudformation describe-stacks \
  --stack-name "$STACK_NAME" \
  | jq -r '[.Stacks[0].Outputs[] | {key: .OutputKey, value: .OutputValue}] | from_entries' | jq -r '.VpcId')

cat > alb-ingress-controller.yaml <<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: alb-ingress-controller
  template:
    metadata:
      labels:
        app.kubernetes.io/name: alb-ingress-controller
    spec:
      containers:
      - name: alb-ingress-controller
        args:
        - --ingress-class=alb
        - --cluster-name=$CLUSTER_NAME
        - --aws-vpc-id=$VPC_ID
        - --aws-region=$REGION
        image: docker.io/amazon/aws-alb-ingress-controller:v1.1.6
      serviceAccountName: alb-ingress-controller
EOF

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

TARGETGROUP_ARN=$(aws --region $REGION --profile $PROFILE \
  elbv2 describe-target-groups | jq -r '.TargetGroups[].TargetGroupArn' | grep $LOADBALANCER_PREFIX)

aws --region $REGION --profile $PROFILE \
  elbv2 describe-target-health \
  --target-group-arn $TARGETGROUP_ARN | jq -r '.TargetHealthDescriptions[].TargetHealth.State'

# Check if all pods are running.
kubectl get pods -o wide
```
