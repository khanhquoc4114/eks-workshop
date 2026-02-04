# Automation
## GitOps
GitOps is a cloud-native operating model that leverages Git as the single source of truth for both infrastructure and application deployments. By applying DevOps best practices—such as version control, collaboration, and compliance—to the entire delivery pipeline, it enables developers to manage complex Kubernetes environments through familiar Git workflows. When a change is pushed to a repository, automated tools like Flux or Argo CD ensure the live state of the cluster matches the desired state defined in code, resulting in faster, more reliable, and highly scalable deployments with reduced operational overhead.
### Flux
Prepare your environment for this section:
``` bash
prepare-environment automation/gitops/flux
```
This will make the following changes to your lab environment:
- Install the AWS Load Balancer controller in the Amazon EKS cluster
- Install the EKS managed addon for the EBS CSI driver

![](../img/automation/2026-02-03-23-36-46.png)
#### Setup Getea 
1. Install Gitea in EKS cluster with Helm:
![](../img/automation/2026-02-03-23-42-00.png)
2. Make sure that Gitea is up and running before proceeding:
![](../img/automation/2026-02-03-23-42-56.png)
3. An SSH key will be needed to interact with Git. The environment preparation for this lab created one, we just need to register it with Gitea:
![](../img/automation/2026-02-03-23-44-17.png)
4. And also need to create the Gitea repository that Flux will use:
![](../img/automation/2026-02-03-23-44-38.png)
5. Finally set up an identity that Git will use for our commits:
![](../img/automation/2026-02-03-23-47-39.png)
#### Cluter bootstrap
Run pre-bootstrap checks to verify that everything is set up correctly. Run the following command for Flux CLI to perform the checks:
![](../img/automation/2026-02-03-23-49-22.png)
Now we can bootstrap Flux on our EKS cluster using a Gitea repository:
![](../img/automation/2026-02-03-23-51-22.png)
![](../img/automation/2026-02-03-23-51-39.png)
Now, let's verify that the bootstrap process completed successfully by running the following command:
![](../img/automation/2026-02-03-23-52-14.png)
#### Deploying an application
First let's remove the existing UI component so we can replace it:
Next, clone the repository we used to bootstrap Flux in the previous section:
![](../img/automation/2026-02-03-23-53-26.png)
Now, let's start populating the Flux repository by creating a directory for our "apps". This directory is designed to contain a sub-directory for each application component:
```bash
mkdir ~/environment/flux/apps
```
![](../img/automation/2026-02-03-23-54-20.png)
Copy this file to the Git repository directory:
![](../img/automation/2026-02-03-23-55-32.png)
Copy this file to the Git repository directory:
![](../img/automation/2026-02-03-23-55-50.png)
Copy the appropriate files to the Git repository directory:
![](../img/automation/2026-02-03-23-56-13.png)
![](../img/automation/2026-02-03-23-56-58.png)
Finally we can push our configuration to Git:
![](../img/automation/2026-02-03-23-57-31.png)
It will take Flux some time to notice the changes in Git and reconcile. You can use the Flux CLI to watch for our new apps kustomization to appear:
![](../img/automation/2026-02-04-00-01-58.png)
Once apps appears as indicated above use Ctrl+C to close the command. You should now have all the resources related to the UI services deployed once more. To verify, run the following commands:
![](../img/automation/2026-02-04-00-09-58.png)
Get the URL from the Ingress resource:
![](../img/automation/2026-02-04-00-10-33.png)
To wait until the load balancer has finished provisioning you can run this command:
![](../img/automation/2026-02-04-00-10-57.png)
And access it in your web browser. You will see the UI from the web store displayed and will be able to navigate around the site as a user.
![](../img/automation/2026-02-04-00-12-23.png)
### ArgoCD
![](../img/automation/2026-02-04-00-33-10.png)
#### Setting up Gitea
Let's install Gitea in our EKS cluster with Helm:
![](../img/automation/2026-02-04-00-38-53.png)
Make sure that Gitea is up and running before proceeding:
![](../img/automation/2026-02-04-00-44-25.png)
An SSH key will be needed to interact with Git. The environment preparation for this lab created one, we just need to register it with Gitea:
![](../img/automation/2026-02-04-00-44-45.png)
And we'll also need to create the Gitea repository that Argo CD will use:
![](../img/automation/2026-02-04-00-54-56.png)
Now we can set up an identity that Git will use for our commits:
![](../img/automation/2026-02-04-00-56-12.png)
And finally let's clone the repository and set up the initial structure:
![](../img/automation/2026-02-04-00-57-27.png)
#### Installing Argo CD
Let's begin by installing Argo CD in our cluster:
![](../img/automation/2026-02-04-00-59-52.png)
For the purpose of this lab, the Argo CD server UI has been configured to be accessible outside of the cluster using a Kubernetes service with a load balancer. To obtain the URL, execute the following commands:
![](../img/automation/2026-02-04-01-00-23.png)
The load balancer will take some time to provision. Use this command to wait until Argo CD responds:
![](../img/automation/2026-02-04-01-02-47.png)
For authentication, the default username is admin and the password is auto-generated. Retrieve the password with the following command:
![](../img/automation/2026-02-04-01-03-20.png)
Use the URL and credentials you just obtained to log in to the Argo CD UI.
![](../img/automation/2026-02-04-01-04-03.png)
![](../img/automation/2026-02-04-01-04-44.png)
To interact with Argo CD using the CLI, you need to authenticate with the Argo CD server:
![](../img/automation/2026-02-04-01-06-02.png)
Finally, register the Git repository with Argo CD to provide access:
![](../img/automation/2026-02-04-01-07-11.png)
#### Deploying an application
First, let's remove the existing sample application from the cluster:
![](../img/automation/2026-02-04-01-08-20.png)
Let's copy this file to our Git directory:
![](../img/automation/2026-02-04-01-09-12.png)
Now we'll push our configuration to the Git repository:
![](../img/automation/2026-02-04-01-09-41.png)
Next, let's create an Argo CD Application configured to use our Git repository:
![](../img/automation/2026-02-04-01-12-22.png)
We can verify that the application has been created:
![](../img/automation/2026-02-04-01-11-25.png)
This application is now visible in the Argo CD UI:
![](../img/automation/2026-02-04-01-12-59.png)
![](../img/automation/2026-02-04-01-16-36.png)
In Argo CD, "out of sync" indicates that the desired state defined in your Git repository doesn't match the actual state in your Kubernetes cluster. Although Argo CD is capable of automated synchronization, for now we'll manually trigger this process:
![](../img/automation/2026-02-04-01-17-28.png)
![](../img/automation/2026-02-04-01-17-44.png)
After a short period, the application should reach the Synced state, with all resources deployed. The UI should look like this:
![](../img/automation/2026-02-04-01-17-59.png)
To verify that all resources related to the UI service are deployed, run the following commands:
![](../img/automation/2026-02-04-01-18-45.png)
#### Updating an application
Let's copy this configuration file to our Git repository directory:
![](../img/automation/2026-02-04-01-19-29.png)
Now we'll commit and push our changes to the Git repository:
![](../img/automation/2026-02-04-01-20-19.png)
At this point, Argo CD will detect that the application state in the repository has changed. You can either click Refresh and then Sync in the Argo CD UI, or use the argocd CLI to synchronize the application:
![](../img/automation/2026-02-04-01-21-00.png)
![](../img/automation/2026-02-04-01-21-28.png)
After synchronization completes, the UI deployment should now have 3 pods running:
![](../img/automation/2026-02-04-01-21-52.png)
To verify that our update was successful, let's check the deployment and pod status:
![](../img/automation/2026-02-04-01-22-21.png)


#### App of app
##### Setup
First, let's copy this foundational App of Apps configuration to our Git directory:
![](../img/automation/2026-02-04-01-26-33.png)
Now, let's commit and push these changes to the Git repository:
![](../img/automation/2026-02-04-01-27-11.png)
Next, we need to create a new Argo CD Application to implement the App of Apps pattern. While doing this, we'll enable Argo CD to automatically synchronize the state in the cluster with the configuration in the Git repository using the --sync-policy automated flag:
![](../img/automation/2026-02-04-01-27-58.png)
![](../img/automation/2026-02-04-01-28-23.png)
##### Adding more workloads 
Let's copy the application chart files to our Git repository directory:
![](../img/automation/2026-02-04-01-29-19.png)
Next, commit and push these changes to the Git repository:
![](../img/automation/2026-02-04-01-29-54.png)
Sync the apps application:
![](../img/automation/2026-02-04-01-30-32.png)
![](../img/automation/2026-02-04-01-30-49.png)
When Argo CD completes the process, all our applications will be in the Synced state as shown in the Argo CD UI:
![](../img/automation/2026-02-04-01-31-55.png)
We should now see a set of new namespaces with each application component deployed:
![](../img/automation/2026-02-04-01-32-15.png)
Let's examine one of the deployed workloads more closely. For example, we can check the carts component:
![](../img/automation/2026-02-04-01-32-36.png)

## Control Planes
### AWS Controllers for Kubernetes (ACK)
AWS Controllers for Kubernetes (ACK) is an open-source framework that allows you to manage AWS managed services directly through the Kubernetes API. Instead of context-switching between the AWS Console, Terraform, or CloudFormation, platform engineers can define resources like Amazon RDS, SQS, or DynamoDB using standard Kubernetes YAML manifests. This "GitOps-friendly" approach treats cloud infrastructure exactly like containerized application code, ensuring that your application's dependencies (e.g., a database) are versioned and deployed alongside the application itself. By leveraging ACK via Helm or Terraform, teams can reduce operational overhead and ensure environment consistency across testing and production.
![](../img/automation/2026-02-04-14-17-59.png)

Prepare your environment for this section:
``` bash
prepare-environment automation/controlplanes/ack
```
This will make the following changes to your lab environment:
- Install the AWS Controllers for DynamoDB in the Amazon EKS cluster
#### Introduction
In this section, since we'll be working with Amazon DynamoDB ACK, we first need to install that ACK controller by using the Helm chart:
``` bash
aws ecr-public get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin public.ecr.aws
helm install ack-dynamodb  \
  oci://public.ecr.aws/aws-controllers-k8s/dynamodb-chart \
  --version=${DYNAMO_ACK_VERSION} \
  --namespace ack-system --create-namespace \
  --set "aws.region=${AWS_REGION}" \
  --set "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"="$ACK_IAM_ROLE" \
  --wait
```

![](../img/automation/2026-02-05-04-44-38.png)

The controller will be running as a deployment in the ack-system namespace:
```bash
kubectl get deployment -n ack-system
```
![](../img/automation/2026-02-05-04-45-09.png)

#### How does ACK works?
To gain deeper insight into the objects and API calls the controller listens for, you can run:
``` bash
kubectl get crd
```
![](../img/automation/2026-02-05-04-45-38.png)
#### Provisioning ACK resources
![](../img/automation/2026-02-05-04-46-04.png)
Now, let's apply these updates to the cluster:
```bash
kubectl kustomize ~/environment/eks-workshop/modules/automation/controlplanes/ack/dynamodb \
  | envsubst | kubectl apply -f-
```
![](../img/automation/2026-02-05-04-46-38.png)
The ACK controllers in the cluster will respond to these new resources and provision the AWS infrastructure we've defined in the manifests. To verify that ACK has created the table, run the following command:
``` bash
kubectl wait table.dynamodb.services.k8s.aws items -n carts --for=condition=ACK.ResourceSynced --timeout=15m
kubectl get table.dynamodb.services.k8s.aws items -n carts -ojson | yq '.status."tableStatus"'
```

![](../img/automation/2026-02-05-04-47-16.png)
Finally, let's confirm that the table has been created using the AWS CLI:
``` bash
aws dynamodb list-tables
```
![](../img/automation/2026-02-05-04-47-42.png)
#### Updating the application
Let's apply this new configuration:
``` bash
kubectl kustomize ~/environment/eks-workshop/modules/automation/controlplanes/ack/app \
  | envsubst | kubectl apply -f-
```

![](../img/automation/2026-02-05-04-52-09.png)

Now we need to restart the carts Pods to pick up our new ConfigMap contents:
``` bash
kubectl rollout restart -n carts deployment/carts
kubectl rollout status -n carts deployment/carts --timeout=40s
```
![](../img/automation/2026-02-05-05-08-22.png)
To verify that the application is working with the new DynamoDB table, we can interact with it through a browser. An NLB has been created to expose the sample application for testing:
``` bash
LB_HOSTNAME=$(kubectl -n ui get service ui-nlb -o jsonpath='{.status.loadBalancer.ingress[*].hostname}{"\n"}')
echo "http://$LB_HOSTNAME"
```
![](../img/automation/2026-02-05-05-09-00.png)
To wait until the load balancer has finished provisioning, you can run this command:
```bash
wait-for-lb $(kubectl get service -n ui ui-nlb -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
```
![](../img/automation/2026-02-05-05-09-33.png)
The UI from the web store displayed and will be able to navigate around the site as a user.
![](../img/automation/2026-02-05-05-10-13.png)
![](../img/automation/2026-02-05-05-11-07.png)
To confirm that these items are also in the DynamoDB table, run:
``` bash
aws dynamodb scan --table-name "${EKS_CLUSTER_NAME}-carts-ack"
```
![](../img/automation/2026-02-05-05-11-57.png)
### Crossplane

Crossplane is an open-source CNCF project that transforms your Kubernetes cluster into a universal control plane. It enables platform teams to assemble infrastructure from multiple vendors and expose higher-level self-service APIs for application teams.

#### Prepare Environment
```bash
prepare-environment automation/controlplanes/crossplane
```
This will make the following changes to your lab environment:
- Install Crossplane and the AWS provider in the Amazon EKS cluster

#### How it works

Crossplane operates with two primary components:
1. **Crossplane controller**: Provides core functionality
2. **Crossplane providers**: Controllers and CRDs to integrate with specific providers (e.g., AWS)

**Step 1:** Verify Crossplane deployments
```bash
kubectl get deployment -n crossplane-system
```

Key components:
- `crossplane` - Core controller
- `crossplane-rbac-manager` - RBAC management
- `upbound-provider-family-aws` - AWS provider
- `upbound-aws-provider-dynamodb` - DynamoDB-specific provider

**Step 2:** Understand the architecture
- **Claims**: Namespace-scoped resources (developer interface)
- **Composite Resources (XR)**: Custom resources representing cloud resources
- **Managed Resources**: Interact directly with AWS API
- **Compositions**: Templates defining how XRs create Managed Resources

![](../img/automation/2026-02-05-03-10-00.png)

#### Managed Resources

**Step 1:** Create DynamoDB table using managed resource
```bash
kubectl kustomize ~/environment/eks-workshop/modules/automation/controlplanes/crossplane/managed \
  | envsubst | kubectl apply -f-
```

**Step 2:** Wait for table to be ready
```bash
kubectl wait tables.dynamodb.aws.upbound.io ${EKS_CLUSTER_NAME}-carts-crossplane \
  --for=condition=Ready --timeout=5m
```

**Step 3:** Verify table status
```bash
kubectl get tables.dynamodb.aws.upbound.io
```

![](../img/automation/2026-02-05-03-11-00.png)

#### Updating the Application

**Step 1:** Update ConfigMap to use new DynamoDB table
```bash
kubectl kustomize ~/environment/eks-workshop/modules/automation/controlplanes/crossplane/app \
  | envsubst | kubectl apply -f-
```

**Step 2:** Restart carts pods
```bash
kubectl rollout restart -n carts deployment/carts
kubectl rollout status -n carts deployment/carts --timeout=40s
```

**Step 3:** Verify application works with DynamoDB
```bash
aws dynamodb scan --table-name "${EKS_CLUSTER_NAME}-carts-crossplane"
```

![](../img/automation/2026-02-05-03-12-00.png)

#### Compositions

Compositions enable platform teams to create opinionated templates for deploying cloud resources with enforced standards (tags, encryption, configurations).

**Step 1:** Create CompositeResourceDefinition (XRD)
XRD defines the schema for your custom API. Developers only need to specify:
- Table name
- Key attributes
- Index name

**Step 2:** Create Composition
Composition defines what resources to create when an XR is created.

**Step 3:** Apply composition and XRD
```bash
kubectl apply -k ~/environment/eks-workshop/modules/automation/controlplanes/crossplane/compositions/composition
```

![](../img/automation/2026-02-05-03-13-00.png)

#### Claims

Claims are lightweight proxy resources for creating XRs. Developers use simple specifications while platform teams control underlying implementation.

**Step 1:** Clean up previous managed resource
```bash
kubectl delete tables.dynamodb.aws.upbound.io --all --ignore-not-found=true
kubectl wait --for=delete tables.dynamodb.aws.upbound.io --all --timeout=5m
```

**Step 2:** Create DynamoDB table using Claim
```bash
cat ~/environment/eks-workshop/modules/automation/controlplanes/crossplane/compositions/claim/claim.yaml \
  | envsubst | kubectl -n carts apply -f -
kubectl wait dynamodbtables.awsblueprints.io ${EKS_CLUSTER_NAME}-carts-crossplane -n carts \
  --for=condition=Ready --timeout=5m
```

**Step 3:** Verify resources created
```bash
kubectl get table
kubectl get DynamoDBTable -n carts
kubectl get XDynamoDBTable
kubectl get composition
```

![](../img/automation/2026-02-05-03-14-00.png)

**Step 4:** Test application and verify data in DynamoDB
```bash
aws dynamodb scan --table-name "${EKS_CLUSTER_NAME}-carts-crossplane"
```

### kro - Kube Resource Orchestrator

kro (Kube Resource Orchestrator) is an open-source Kubernetes operator that enables you to define custom APIs through ResourceGraphDefinitions (RGDs). It uses CEL (Common Expression Language) to handle dependencies, conditional logic, and value passing between resources. kro intelligently analyzes resource relationships to determine the correct deployment order.

#### Prepare your environment for this section:

``` bash 
prepare-environment automation/controlplanes/kro
``` 
This will make the following changes to your lab environment:
- Install the AWS Controllers for Kubernetes controllers for EKS, IAM and DynamoDB
- Install the AWS Load Balancer Controller
- Create an Ingress resource for the UI workload

#### Introduction

**Step 1:** Install kro using Helm
```bash
helm install kro oci://ghcr.io/kro-run/kro/kro \
  --version=${KRO_VERSION} \
  --namespace kro-system --create-namespace \
  --wait
```
![](../img/automation/2026-02-05-05-24-17.png)

**Step 2:** Verify kro controller is running
```bash
kubectl get deployment -n kro-system
```
![](../img/automation/2026-02-05-05-25-09.png)
**Step 3:** Verify CRDs installed
```bash
kubectl get crd | grep kro
```

![](../img/automation/2026-02-05-05-25-29.png)

#### How kro works

When you create a ResourceGraphDefinition, kro:
1. Registers a new Custom API based on the RGD schema
2. Processes resource instances when developers create custom API instances
3. Evaluates CEL expressions for conditions and value passing
4. Analyzes dependencies and determines optimal deployment order
5. Creates managed resources in correct order
6. Maintains relationships and lifecycle management

#### Creating resources with kro

**Step 1:** Clean up existing carts deployment
```bash
kubectl delete all --all -n carts
```
![](../img/automation/2026-02-05-05-26-30.png)
**Step 2:** Apply WebApplication ResourceGraphDefinition
```bash
kubectl apply -f ~/environment/eks-workshop/modules/automation/controlplanes/kro/rgds/webapp-rgd.yaml
```
![](../img/automation/2026-02-05-05-26-53.png)
**Step 3:** Verify CRD created
```bash
kubectl get crd webapplications.kro.run
```

![](../img/automation/2026-02-05-05-27-17.png)

**Step 4:** Deploy carts using WebApplication API
```bash
kubectl apply -f ~/environment/eks-workshop/modules/automation/controlplanes/kro/app/carts.yaml
```

![](../img/automation/2026-02-05-05-27-47.png)

**Step 5:** Wait for sync
```bash
kubectl wait webapplication/carts -n carts \
  --for=condition=InstanceSynced=True --timeout=120s
```
![](../img/automation/2026-02-05-05-28-42.png)
**Step 6:** Verify resources
```bash
kubectl get webapplication -n carts
kubectl get all -n carts
```

![](../img/automation/2026-02-05-05-29-29.png)

#### Provisioning cloud resources

Compose WebApplicationDynamoDB RGD that builds on base WebApplication template to add DynamoDB storage.

**Step 1:** Delete previous kro instance
```bash
kubectl delete webapplication.kro.run/carts -n carts
kubectl get all -n carts
```
![](../img/automation/2026-02-05-05-30-56.png)
**Step 2:** Apply WebApplicationDynamoDB RGD
```bash
kubectl apply -f ~/environment/eks-workshop/modules/automation/controlplanes/kro/rgds/webapp-dynamodb-rgd.yaml
```
![](../img/automation/2026-02-05-05-31-23.png)
This RGD:
- Creates custom `WebApplicationDynamoDB` API
- Provisions DynamoDB table with ACK
- Creates IAM roles and policies
- Configures EKS Pod Identity

**Step 3:** Verify CRD created
```bash
kubectl get crd webapplicationdynamodbs.kro.run
```
![](../img/automation/2026-02-05-05-31-45.png)
**Step 4:** Deploy carts with DynamoDB
```bash
kubectl kustomize ~/environment/eks-workshop/modules/automation/controlplanes/kro/app \
  | envsubst | kubectl apply -f-
```
![](../img/automation/2026-02-05-05-33-48.png)
**Step 5:** Wait for sync
```bash
kubectl wait webapplicationdynamodb/carts -n carts \
  --for=condition=InstanceSynced=True --timeout=120s
```
![](../img/automation/2026-02-05-05-35-24.png)
**Step 6:** Verify DynamoDB table created
```bash
kubectl wait table.dynamodb.services.k8s.aws items -n carts --for=condition=ACK.ResourceSynced --timeout=15m
kubectl get table.dynamodb.services.k8s.aws items -n carts -ojson | yq '.status."tableStatus"'
aws dynamodb list-tables
```
![](../img/automation/2026-02-05-05-36-07.png)


**Step 7:** Test application with DynamoDB
To verify that the component is working with the new DynamoDB table, we can interact with it through a browser. An NLB has been created to expose the sample application for testing:
![](../img/automation/2026-02-05-05-37-39.png)

Go to site and but some items
![](../img/automation/2026-02-05-05-38-51.png)
![](../img/automation/2026-02-05-05-39-04.png)
```bash
aws dynamodb scan --table-name "eks-workshop-carts-kro"
```
![](../img/automation/2026-02-05-05-40-11.png)

