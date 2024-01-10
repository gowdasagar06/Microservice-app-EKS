eksctl create cluster --name test-cluster --region ap-south-1
export cluster_name=test-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5) 
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
eksctl create iamserviceaccount \
  --cluster=test-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::528462248584:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system  --set clusterName=test-cluster  --set serviceAccount.create=false  --set serviceAccount.name=aws-load-balancer-controller  --set region=ap-south-1  --set vpcId=vpc-01c2c479082121ac5

kubectl get deployment -n kube-system aws-load-balancer-controller
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster test-cluster \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
eksctl create addon --name aws-ebs-csi-driver --cluster test-cluster --service-account-role-arn arn:aws:iam::528462248584:role/AmazonEKS_EBS_CSI_DriverRole --force  
cd Microservice-app-EKS/EKS/helm/  
kubectl create ns robot-shop
helm install robot-shop --namespace robot-shop .
kubectl get pods -n robot-shop
kubectl get svc -n robot-shop
kubectl apply -f ingress.yaml
