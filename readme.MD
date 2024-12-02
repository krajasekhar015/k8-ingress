# Creating Ingress Controller

Go to the official website
```
https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/
```

> Make sure you run the below commands out of the git repository

1. Create an IAM OIDC provider
```
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster expense \
    --approve
```

2. Download an IAM policy for the LBC using one of the following commands:
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.10.0/docs/install/iam_policy.json
```

3. Create an IAM policy named AWSLoadBalancerControllerIAMPolicy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```
- Take note of the policy ARN that's returned
- AWS call ingress controller as LoadBalancerController

4. Create an IAM role and Kubernetes ServiceAccount for the LBC. Use the ARN from the previous step.
```
eksctl create iamserviceaccount \
--cluster=expense \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::315069654700:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region us-east-1 \
--approve
```
- Here, `315069654700` is AWS Account ID

- ALB is an external component. If we want to connect the EKS and ALB properly, then we need to install helm drivers

5. Add the EKS chart repo to Helm
```
helm repo add eks https://aws.github.io/eks-charts
```
6. Helm install command for clusters with IRSA:
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=expense --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

