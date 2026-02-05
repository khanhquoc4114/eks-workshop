# Networking

## Overview

Understanding Kubernetes networking is critical for efficiently operating your cluster and applications. This guide covers Pod networking, service networking, Amazon VPC CNI configurations, VPC Lattice, and EKS Hybrid Nodes.

Amazon EKS uses Amazon VPC to provide networking capabilities to worker nodes and Kubernetes Pods. The cluster consists of an AWS-managed VPC hosting the Kubernetes control plane and a customer-managed VPC hosting worker nodes and AWS infrastructure.

---

## 1. Amazon VPC CNI

### Overview

Amazon EKS officially supports the Amazon VPC CNI plugin to implement Kubernetes Pod networking. The VPC CNI provides native integration with AWS VPC and works in underlay mode where Pods and hosts share the same network layer.

Worker nodes connect to the EKS control plane through the EKS public endpoint or EKS-managed elastic network interfaces (ENIs). The VPC CNI assigns Pod IP addresses from VPC subnets, making them consistent from both cluster and VPC perspectives.


---

## 2. Custom Networking

### Overview

By default, Amazon VPC CNI assigns Pods IP addresses from the primary subnet. Custom networking solves IP exhaustion issues by assigning Pod IPs from secondary VPC address spaces using ENIConfig custom resources.


### Steps

**Step 1:** Prepare environment
```bash
prepare-environment networking/custom-networking
```

This creates:
- Secondary CIDR range attached to VPC
- Three additional subnets from secondary CIDR

**Step 2:** Review VPC architecture
```bash
aws ec2 describe-vpcs --vpc-ids $VPC_ID
```

Expected output shows:
- Primary CIDR: `10.42.0.0/16`
- Secondary CIDR: `100.64.0.0/16`

**Step 3:** List subnets
```bash
aws ec2 describe-subnets --filters "Name=tag:created-by,Values=eks-workshop-v2" \
  --query "Subnets[*].CidrBlock"
```

Subnets include:
- Public subnets (primary CIDR)
- Private subnets (primary CIDR)
- Secondary private subnets (secondary CIDR)


**Step 4:** Enable custom networking
```bash
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
```

**Step 5:** Create ENIConfig resources
```bash
kubectl kustomize ~/environment/eks-workshop/modules/networking/custom-networking/provision \
  | envsubst | kubectl apply -f-
```

**Step 6:** Verify ENIConfig creation
```bash
kubectl get ENIConfigs
```

**Step 7:** Configure automatic ENIConfig assignment
```bash
kubectl set env daemonset aws-node -n kube-system ENI_CONFIG_LABEL_DEF=topology.kubernetes.io/zone
```

**Step 8:** Provision new node group
```bash
aws eks create-nodegroup --region $AWS_REGION \
  --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name custom-networking \
  --instance-types t3.medium --node-role $CUSTOM_NETWORKING_NODE_ROLE \
  --subnets $PRIMARY_SUBNET_1 $PRIMARY_SUBNET_2 $PRIMARY_SUBNET_3 \
  --labels type=customnetworking \
  --scaling-config minSize=1,maxSize=1,desiredSize=1
```

**Step 9:** Wait for node group creation
```bash
aws eks wait nodegroup-active --cluster-name $EKS_CLUSTER_NAME --nodegroup-name custom-networking
```

**Step 10:** Verify new nodes
```bash
kubectl get nodes -L eks.amazonaws.com/nodegroup
```

New nodes will use secondary CIDR for Pod IP allocation.

**Step 11:** Deploy sample application
Deploy workloads to the new node group and verify they receive IPs from the secondary CIDR range.

---

## 3. Prefix Delegation

### Overview

Amazon VPC CNI assigns network prefixes to EC2 network interfaces to increase the number of available IP addresses and boost pod density per node. With prefix delegation, VPC CNI assigns /28 (16 IP addresses) IPv4 prefixes instead of individual secondary IP addresses.



### Steps

**Step 1:** Prepare environment
```bash
prepare-environment networking/prefix
```

**Step 2:** Verify VPC CNI installation
```bash
kubectl get pods --selector=k8s-app=aws-node -n kube-system
```

**Step 3:** Check CNI version
```bash
kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2
```

Version must be 1.9.0 or later.

**Step 4:** Confirm prefix delegation is enabled
```bash
kubectl get ds aws-node -o yaml -n kube-system | yq '.spec.template.spec.containers[].env'
```

Look for `ENABLE_PREFIX_DELEGATION: "true"`

**Step 5:** Verify prefixes assigned to nodes
```bash
aws ec2 describe-instances --filters "Name=tag-key,Values=eks:cluster-name" \
  "Name=tag-value,Values=${EKS_CLUSTER_NAME}" \
  --query 'Reservations[*].Instances[].{InstanceId: InstanceId, Prefixes: NetworkInterfaces[].Ipv4Prefixes[]}'
```

**Step 6:** Review prefix allocation
Each node should have one or more /28 IPv4 prefixes assigned.

**Step 7:** Monitor pod density
With prefix delegation, nodes can support significantly more pods than with secondary IP mode.

---

## 4. Network Policies

### Overview

Kubernetes Network Policies enable defining and enforcing rules on traffic flow between pods, namespaces, and IP blocks. They act as a virtual firewall for segmenting and securing the cluster.


### Steps

**Step 1:** Prepare environment
```bash
prepare-environment networking/network-policies
```

**Step 2:** Review sample application architecture


Each component runs in its own namespace: ui, catalog, orders, etc.

**Step 3:** Verify current connectivity (no restrictions)
```bash
kubectl exec deployment/catalog -n catalog -- curl -s http://checkout.checkout/health
```

Should return success response.

**Step 4:** Implement default deny egress policy
```bash
kubectl apply -n ui -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/default-deny.yaml
```

**Step 5:** Verify egress blocked
```bash
kubectl exec deployment/ui -n ui -- curl -s http://catalog.catalog/health --connect-timeout 5
```

Should timeout.

**Step 6:** Apply selective egress policy for UI
```bash
kubectl apply -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-ui-egress.yaml
```

This allows:
- Traffic to all service components
- Traffic to kube-system namespace for DNS

**Step 7:** Verify UI can access catalog service
```bash
kubectl exec deployment/ui -n ui -- curl http://catalog.catalog/health
```

Should succeed.

**Step 8:** Verify UI cannot access database
```bash
kubectl exec deployment/ui -n ui -- curl -v telnet://catalog-mysql.catalog:3306 --connect-timeout 5
```

Should timeout.

**Step 9:** Apply ingress policy for catalog service
```bash
kubectl apply -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-catalog-ingress-webservice.yaml
```

Allows traffic only from UI namespace.

**Step 10:** Verify UI can access catalog
```bash
kubectl exec deployment/ui -n ui -- curl -v catalog.catalog/health --connect-timeout 5
```

**Step 11:** Verify orders cannot access catalog
```bash
kubectl exec deployment/orders -n orders -- curl -v catalog.catalog/health --connect-timeout 5
```

Should timeout.

**Step 12:** Apply ingress policy for catalog database
```bash
kubectl apply -f ~/environment/eks-workshop/modules/networking/network-policies/apply-network-policies/allow-catalog-ingress-db.yaml
```

Allows database access only from catalog service pods.

**Step 13:** Test database isolation
```bash
kubectl exec deployment/orders -n orders -- curl -v telnet://catalog-mysql.catalog:3306 --connect-timeout 5
```

Should timeout, confirming database is protected.

---

## 5. Security Groups for Pods

### Overview

Security Groups for Pods allow using Amazon EC2 security groups to define rules governing inbound and outbound network traffic to and from specific pods. This provides pod-level network security instead of node-level.


### Steps

**Step 1:** Prepare environment
```bash
prepare-environment networking/securitygroups-for-pods
```

This creates:
- Amazon RDS instance
- EC2 security group for RDS access

**Step 2:** Review current catalog setup
```bash
kubectl -n catalog get pod
```

Shows catalog service and MySQL database pods.

**Step 3:** Check catalog database endpoint
```bash
kubectl -n catalog exec deployment/catalog -- env \
  | grep RETAIL_CATALOG_PERSISTENCE_ENDPOINT
```

Currently points to in-cluster MySQL.

**Step 4:** Verify RDS security group
```bash
export CATALOG_SG_ID=$(aws ec2 describe-security-groups \
    --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=$EKS_CLUSTER_NAME-catalog \
    --query "SecurityGroups[0].GroupId" --output text)
aws ec2 describe-security-groups --group-ids $CATALOG_SG_ID
```

Security group allows:
- Inbound HTTP API traffic on port 8080
- All egress traffic
- Access to RDS database

**Step 5:** Create SecurityGroupPolicy
```bash
kubectl kustomize ~/environment/eks-workshop/modules/networking/securitygroups-for-pods/sg \
  | envsubst | kubectl apply -f-
```

This maps the security group to catalog service pods.

**Step 6:** Recycle catalog pods
```bash
kubectl delete pod -n catalog -l app.kubernetes.io/component=service
```

**Step 7:** Wait for rollout
```bash
kubectl rollout status -n catalog deployment/catalog --timeout 30s
```

**Step 8:** Verify RDS connection
```bash
kubectl -n catalog logs deployment/catalog
```

Should show connection to RDS endpoint.

**Step 9:** Inspect pod ENI
Catalog pods now have dedicated branch ENIs with the security group attached.

**Step 10:** Verify trunk interface on node
Check node for `aws-k8s-trunk-eni` interface created by VPC Resource Controller.

**Step 11:** Test connectivity
Catalog pods can now access RDS while other pods cannot due to security group restrictions.

---

## 6. Amazon VPC Lattice

### Overview

Amazon VPC Lattice is an application layer networking service providing consistent connectivity, security, and monitoring for service-to-service communication across VPCs and accounts.


Benefits:
- Increased developer productivity
- Improved security posture
- Better application scalability and resilience
- Deployment flexibility with heterogeneous infrastructure

### Steps

**Step 1:** Prepare environment
```bash
prepare-environment networking/vpc-lattice
```

This installs:
- IAM role for Gateway API controller
- AWS Load Balancer Controller

**Step 2:** Configure security group for VPC Lattice
```bash
CLUSTER_SG=$(aws eks describe-cluster --name $EKS_CLUSTER_NAME --output json| jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')
PREFIX_LIST_ID=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID}}],IpProtocol=-1"
PREFIX_LIST_ID_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.ipv6.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID_IPV6}}],IpProtocol=-1"
```

**Step 3:** Install Gateway API CRDs
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

**Step 4:** Install VPC Lattice controller
```bash
aws ecr-public get-login-password --region us-east-1 \
  | helm registry login --username AWS --password-stdin public.ecr.aws
helm install gateway-api-controller \
    oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
    --version=v${LATTICE_CONTROLLER_VERSION} \
    --create-namespace \
    --set=aws.region=${AWS_REGION} \
    --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$LATTICE_IAM_ROLE" \
    --set=defaultServiceNetwork=${EKS_CLUSTER_NAME} \
    --namespace gateway-api-controller \
    --wait
```

**Step 5:** Verify controller deployment
```bash
kubectl get deployment -n gateway-api-controller
```

**Step 6:** Create GatewayClass
```bash
kubectl apply -f ~/environment/eks-workshop/modules/networking/vpc-lattice/controller/gatewayclass.yaml
```

**Step 7:** Create Gateway (Service Network)
```bash
cat ~/environment/eks-workshop/modules/networking/vpc-lattice/controller/eks-workshop-gw.yaml \
  | envsubst | kubectl apply -f -
```

**Step 8:** Verify Gateway creation
```bash
kubectl get gateway -n checkout
```

**Step 9:** Wait for Gateway to be programmed
```bash
kubectl wait --for=condition=Programmed gateway/${EKS_CLUSTER_NAME} -n checkout
```

**Step 10:** View Service Network in AWS Console
Navigate to VPC Console → Lattice → Service Networks


**Step 11:** Configure routes
Create HTTPRoute resources to define traffic routing rules.

**Step 12:** Test routing
Verify traffic flows through VPC Lattice according to defined routes.

---

## 7. EKS Hybrid Nodes

### Overview

Amazon EKS Hybrid Nodes unifies Kubernetes management across cloud, on-premises, and edge environments, providing flexibility to run workloads anywhere while maintaining centralized control.


### Steps

**Step 1:** Prepare environment
```bash
prepare-environment networking/eks-hybrid-nodes
```

This creates:
- Transit Gateway connection to simulated remote network
- EC2 instance simulating on-premises node

**Step 2:** Create SSM hybrid activation
```bash
export ACTIVATION_JSON=$(aws ssm create-activation \
--default-instance-name hybrid-ssm-node \
--iam-role $HYBRID_ROLE_NAME \
--registration-limit 1 \
--region $AWS_REGION)
export ACTIVATION_ID=$(echo $ACTIVATION_JSON | jq -r ".ActivationId")
export ACTIVATION_CODE=$(echo $ACTIVATION_JSON | jq -r ".ActivationCode")
```

**Step 3:** Create NodeConfig
```bash
cat ~/environment/eks-workshop/modules/networking/eks-hybrid-nodes/nodeconfig.yaml \
| envsubst > nodeconfig.yaml
```

**Step 4:** Copy NodeConfig to hybrid node
```bash
mkdir -p ~/.ssh/
ssh-keyscan -H $HYBRID_NODE_IP &> ~/.ssh/known_hosts
scp -i private-key.pem nodeconfig.yaml ubuntu@$HYBRID_NODE_IP:/home/ubuntu/nodeconfig.yaml
```

**Step 5:** Install nodeadm dependencies
```bash
ssh -i private-key.pem ubuntu@$HYBRID_NODE_IP \
"sudo nodeadm install $EKS_CLUSTER_VERSION --credential-provider ssm"
```

Installs: containerd, kubelet, kubectl, AWS SSM components

**Step 6:** Initialize hybrid node
```bash
ssh -i private-key.pem ubuntu@$HYBRID_NODE_IP \
"sudo nodeadm init -c file://nodeconfig.yaml"
```

**Step 7:** Verify node joined cluster
```bash
kubectl get nodes
```

Node appears with `mi-` prefix (SSM credential provider) but in `NotReady` state.

**Step 8:** Add Cilium Helm repo
```bash
helm repo add cilium https://helm.cilium.io/
```

**Step 9:** Install Cilium CNI
```bash
helm install cilium cilium/cilium \
--version 1.17.1 \
--namespace cilium \
--create-namespace \
--values ~/environment/eks-workshop/modules/networking/eks-hybrid-nodes/cilium-values.yaml
```

Configuration includes:
- Node affinity for hybrid nodes only
- Cluster-pool IPAM mode
- Dedicated CIDR (10.53.0.0/16) for hybrid node pods
- /25 subnets per node (128 IPs)

**Step 10:** Wait for node to become Ready
```bash
kubectl wait --for=condition=Ready nodes --all --timeout=2m
```

**Step 11:** Verify hybrid node status
```bash
kubectl get nodes
```

Hybrid node should now show `Ready` status.

**Step 12:** Route traffic to hybrid node
Deploy workloads with node selectors targeting hybrid nodes.

**Step 13:** Test cloud bursting
Scale applications to utilize both cloud and hybrid nodes.

---