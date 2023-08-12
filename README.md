# AWS Pod EIP Controller

The AWS Pod EIP Controller (PEC) offers a function to automatically allocate and release Elastic IPs via annotations. It also enables automatic association of allocated EIPs with the PODs and provides the ability to add EIPs to Shied protection via annotations. This feature enhances security by allowing for better control over IP addresses used in AWS resources.

## Prerequisites

* Have an EKS cluster
* The Pod that wants to add the EIP runs on the nodes in the public subnets

## Build and push image to ECR

1. Set the current account and region

```shell
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)

export AWS_REGION=<region-code>
```

2. Build and push EPC images to ECR

```shell
aws ecr create-repository --repository-name aws-pod-eip-controller

docker buildx build -t aws-pod-eip-controller . --platform linux/amd64 --build-arg TARGETOS=linux --build-arg TARGETARCH=amd64

aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

docker tag aws-pod-eip-controller:latest ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/aws-pod-eip-controller

docker push ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/aws-pod-eip-controller
```

## Configure IAM

### IAM roles for service accounts (IRSA)

The reference IAM policies contain the following permissive configuration:

```yaml
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ec2:ReleaseAddress",
                "ec2:DisassociateAddress",
                "ec2:DescribeAddresses",
                "shield:CreateProtection",
                "ec2:DescribeNetworkInterfaces",
                "shield:DescribeProtection",
                "shield:DescribeSubscription",
                "ec2:CreateTags",
                "shield:DeleteProtection",
                "ec2:AssociateAddress",
                "ec2:AllocateAddress"
            ],
            "Resource": "*"
        }
    ]
}
```

2. Create an IAM OIDC provider. You can skip this step if you already have one for your cluster.

```yaml
eksctl utils associate-iam-oidc-provider \
    --region <region-code> \
    --cluster <your-cluster-name> \
    --approve
```

3. Save the IAM policy to iam-policy.json
4. Create an IAM policy named AWSPodEIPControllerIAMPolicy

```yaml
aws iam create-policy \
    --policy-name AWSPodEIPControllerIAMPolicy \
    --policy-document file://iam-policy.json
```

Take note of the policy ARN that's returned.

5. Create an IAM role and Kubernetes ServiceAccount for the PEC. Use the ARN from the previous step.

```shell
eksctl create iamserviceaccount \
    --cluster=<cluster-name> \
    --namespace=kube-system \
    --name=aws-pod-eip-controller \
    --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSPodEIPControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region <region-code> \
    --approve
```

## Deploy EPC

Replace the image address in template.yaml with the address of the ECR image

```shell
kubectl apply -f template.yaml
```

## Annotations

Name|Type|Default|Location
-|-|-|-
service.beta.kubernetes.io/aws-eip-pod-controller-type|auto|N/A|pod
service.beta.kubernetes.io/aws-eip-pod-controller-shield|advanced|N/A|pod

## environment variables

Name|Type|Default|Describetion
-|-|-|-
VPC_ID|string|N/A|need to provide when debugging locally or deploying in fargate
REGION|string|N/A|need to provide when debugging locally or deploying in fargate
NAMESPACE|string|''|Which namespace to listen on only, Empty to listen to all
LOG_LEVEL|string|info|log level: panic fatal error warn info debug trace
CHANNEL_SIZE|int|20|number of pipelines

## Demo

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-user
  namespace: nginx-demo-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: nginx-demo-ns
  name: app-nginx-demo
  labels:
    app.kubernetes.io/name: app-nginx-demo
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-nginx-demo
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: app-nginx-demo
      annotations:
        service.beta.kubernetes.io/aws-eip-pod-controller-type: auto
        service.beta.kubernetes.io/aws-eip-pod-controller-shield: advanced
    spec:
      serviceAccountName: nginx-user
      containers:
      - image: nginx:1.20
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/nginx/nginx.conf
          readOnly: true
          name: nginx-conf
          subPath: nginx.conf
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-conf
          items:
            - key: nginx.conf
              path: nginx.conf
```

```shell
kubectl apply -f nginx.demo.yaml
```