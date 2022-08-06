# ekscluster-creation-with-configfile-and-install-alb-ingress-controller-

Create an ec2-instance and attach admin role to that instance
-------------------------------------------------------------

Install kubectl (older version 1.14.6)
--------------------------------------
curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.14.6/2019-08-22/bin/linux/amd64/kubectl
chmod +x ./kubectl
cp kubectl /usr/bin

Install eksctl:
---------------
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/bin
eksctl version


vi cluster.yml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: eksdemo
  region: us-east-1
  version: "1.21"

managedNodeGroups:
  - name: workers
    instanceType: t2.medium
    privateNetworking: true
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
    volumeSize: 10
    ssh:
      allow: true
      publicKeyName: 8ambatch
    labels: {role: worker}
    tags:
      nodegroup-role: worker
    iam:
      withAddonPolicies:
        externalDNS: true
        certManager: true
        autoScaler: true


Create cluster
--------------
eksctl create cluster -f cluster.yml

(Now Cluster will be created with master and nodegroup)

alias k=kubectl
k get nodes





Install ALB Ingress Controller
------------------------------

# Create ClusterRole, ClusterRoleBinding & ServiceAccount
    ServiceAccount name "alb-ingress-controller"
    (within k8's we are creating this clusterrole so this clusterrole will be attached to serviceaccount using clusterrolebinding)
---------------------------------------------------------------------------------------------------------------------------------
k apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/rbac-role.yaml

List Service Accounts
-----------------------
k get sa -n kube-system



Create policy named "alb-ingress-controller-policy" using below json file in IAM
--------------------------------------------------------------------------------
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "wafv2:AssociateWebACL",
                "ec2:AuthorizeSecurityGroupIngress",
                "elasticloadbalancing:ModifyListener",
                "ec2:DescribeInstances",
                "wafv2:GetWebACLForResource",
                "elasticloadbalancing:RegisterTargets",
                "iam:ListServerCertificates",
                "wafv2:GetWebACL",
                "elasticloadbalancing:SetIpAddressType",
                "ec2:DescribeInternetGateways",
                "elasticloadbalancing:DeleteLoadBalancer",
                "elasticloadbalancing:SetWebAcl",
                "waf-regional:GetWebACLForResource",
                "elasticloadbalancing:DescribeLoadBalancers",
                "acm:GetCertificate",
                "waf-regional:GetWebACL",
                "shield:DescribeSubscription",
                "elasticloadbalancing:CreateRule",
                "ec2:DescribeAccountAttributes",
                "elasticloadbalancing:AddListenerCertificates",
                "elasticloadbalancing:ModifyTargetGroupAttributes",
                "waf:GetWebACL",
                "iam:GetServerCertificate",
                "wafv2:DisassociateWebACL",
                "ec2:CreateTags",
                "shield:GetSubscriptionState",
                "ec2:ModifyNetworkInterfaceAttribute",
                "elasticloadbalancing:CreateTargetGroup",
                "elasticloadbalancing:DeregisterTargets",
                "ec2:RevokeSecurityGroupIngress",
                "elasticloadbalancing:DescribeLoadBalancerAttributes",
                "acm:DescribeCertificate",
                "elasticloadbalancing:DescribeTargetGroupAttributes",
                "shield:CreateProtection",
                "elasticloadbalancing:ModifyRule",
                "elasticloadbalancing:AddTags",
                "elasticloadbalancing:DescribeRules",
                "ec2:DescribeSubnets",
                "elasticloadbalancing:ModifyLoadBalancerAttributes",
                "waf-regional:AssociateWebACL",
                "ec2:DescribeAddresses",
                "tag:GetResources",
                "ec2:DeleteTags",
                "shield:DescribeProtection",
                "shield:DeleteProtection",
                "elasticloadbalancing:RemoveListenerCertificates",
                "tag:TagResources",
                "elasticloadbalancing:RemoveTags",
                "elasticloadbalancing:CreateListener",
                "ec2:DescribeNetworkInterfaces",
                "elasticloadbalancing:DescribeListeners",
                "ec2:CreateSecurityGroup",
                "acm:ListCertificates",
                "elasticloadbalancing:DescribeListenerCertificates",
                "ec2:ModifyInstanceAttribute",
                "elasticloadbalancing:DeleteRule",
                "cognito-idp:DescribeUserPoolClient",
                "ec2:DescribeInstanceStatus",
                "elasticloadbalancing:DescribeSSLPolicies",
                "elasticloadbalancing:CreateLoadBalancer",
                "waf-regional:DisassociateWebACL",
                "ec2:DescribeTags",
                "elasticloadbalancing:DescribeTags",
                "elasticloadbalancing:SetSubnets",
                "elasticloadbalancing:DeleteTargetGroup",
                "ec2:DescribeSecurityGroups",
                "iam:CreateServiceLinkedRole",
                "ec2:DescribeVpcs",
                "ec2:DeleteSecurityGroup",
                "elasticloadbalancing:DescribeTargetHealth",
                "elasticloadbalancing:SetSecurityGroups",
                "elasticloadbalancing:DescribeTargetGroups",
                "shield:ListProtections",
                "elasticloadbalancing:ModifyTargetGroup",
                "elasticloadbalancing:DeleteListener"
            ],
            "Resource": "*"
        }
    ]
}


  Add Iam-Oidc-Providers (acts like a bridge between aws environment and k8's cluster)
--------------------------------------------------------------------------------------
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo \
--approve


Create Role named "iamserviceaccount" and attach policy to a Role: (This created role will be attached to serviceaccount)
-------------------------------------------------------------------------------------------------------------------------
eksctl create iamserviceaccount \
    --region us-east-1 \
    --name alb-ingress-controller \
    --namespace kube-system \
    --cluster eksdemo \
    --attach-policy-arn arn:aws:iam::200808950444:policy/alb-ingress-controller-policy \
--override-existing-serviceaccounts \
    --approve


To describe our serviceaccount(sa) alb-ingress-controller (our created role is attached to our service account)
---------------------------------------------------------------------------------------------------------------
k describe sa alb-ingress-controller -n kube-system



To deploy alb-ingress-controller
--------------------------------
k apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/alb-ingress-controller.yaml


Verify Deployment
-----------------
k get deploy -n kube-system

k logs <alb-ingress-controller-podname> -n kube-system




Edit ALB Ingress Controller Manifest:
--------------------------------------
k edit deploy alb-ingress-controller -n kube-system



Replaced cluster-name with our cluster-name eksdemo
---------------------------------------------------
spec:
  containers:
  - args:
    - --ingress-class=alb
    - --cluster-name=eksdemo


Verify Deployment
-----------------
k get deploy -n kube-system


Verify our ALB Ingress Controller is running
--------------------------------------------
# Verify if alb-ingress-controller pod is running
k get pods -n kube-system






##Deploying springboot application with ALB-Ingress-controller
--------------------------------------------------------------
vi deployment.yml
-----------------
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-deployment
  labels:
    app: springboot
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 1
  selector:
    matchLabels:
      app: springboot
  template:
    metadata:
      labels:
        app: springboot
    spec:
      containers:
      - name: springboot-deployment
        image: sarath750/springboot-hello:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080

k apply -f deployment.yml

k get nodes

k get pods


vi service.yml
--------------
---
apiVersion: v1
kind: Service
metadata:
  name: springboot-service
spec:
  type: LoadBalancer
  selector:
    app: springboot
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080

k apply -f servie.yml

get the DNS name from load-balancer 
<DNS name:8080>



vi ingress.yml
--------------
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: springboot-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: springboot-service
          servicePort:  8080

k apply -f ingress.yml

k get ingress
(address not present in the ingress so we need to go and check inside the alb-ingress-controller pod)


k get pods -n kube-system

k logs <alb-ingress-controller pod name> -n kube-system
( Subnets must contains these tags: 'kubernetes.io/cluster/eksdemo': ['shared' or 'owned'] and 'kubernetes.io/role/elb':)
--> add these tags only to public subnets inside vpc--> subnets --> <eksctl-eksdemo-cluster/SubnetPublicUSEAST1F> --> managetags (kubernetes.io/cluster/eksdemo/owned)

ALB Ingress Controller will create ALB. It can be seen in "LoadBalancer" section. the ingress will be running with the 80 Portnumber.

<ALB DNS Name>:80  (browse)
Now we are able to see the output of the application at the ALB level, but if we want to see the out of the application at DNS level Route53 comes into picture.



Route53
-------
If any user's request from outside needs to communicate with our deployed application in k8's, Route53 helps the user to connect the user request with our deployed application in k8's. 
Amazon Route 53 is a highly available and scalable Domain Name System (DNS) web service. Route 53 connects user requests to internet applications running on AWS or on-premises.

create hosted-zone and place nameservers in our created domain. so Route53 can communicate with our domain.

create recordname with A name record in this hosted-zone, so it can communicate with our application through ALB.


 Create a hosted zone in route53 with the domainname(domain which is purchased in godaddy). once this hosted zone is created it will provide nameservers. place this 4 nameservers inside our domain in godaddy.

when this 4 nameservers are placed in our domain created in godaddy, then route53 can communicate with our domain.

Communicating with our deployed application
-------------------------------------------
inside that created hosted zone, create a record name(springboot) in route53. this created record will be communicating with our application through ALB.

user's request(DNS)
-------------------
springboot.awstrainer.com 

output of our application
-------------------------
Greetings from Springboot..!!! 

recordname - springboot
domain - awstrainer.com


Now we are able to see the output of the application at the DNS level.



our URL springboot.awstrainer.com is not secure(80 PN), we need to make it secure(443 PN). 

Certificate Manager
-------------------
open certificate manager service in aws and create a certificate by providing full domain name.

create a record with c name record in route53.

http(80) --> https(443)

add other annotations as well inside ingress.yml file.
ssl-redirect annotation: it will redirect from 8080 to 443.
hostname annotation: <springboot.awstrainer.com>
certificate arn annotation: <keep certificate arn>

Now our DNS is secured.
unsecured --> secured
http(80) --> https(443)


