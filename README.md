# BottleRocket EKS Cluster via eksctl

AWS CLI: https://aws.amazon.com/cli/ \
kubectl: https://kubernetes.io/docs/reference/kubectl/ \
eksctl: https://eksctl.io/ \
Bottlerocket: https://aws.amazon.com/bottlerocket/ https://github.com/bottlerocket-os/bottlerocket \
Helm: https://helm.sh/

Instructions on how to create an EKS cluster with: 
* Automatic persistent volume provisioning using EBS volumes
* Worldpay internal DNS
* Cluster autoscaling
* K8s dashboard
* Ingress controller
* Prometheus and Grafana for monitoring
* GitHub Actions self-hosted runner operator
* Instana agent

All applications installed using Helm for ease of configuration and simplified upgrades.

## Prerequisites
Install:
1. AWS CLI
2. kubectl
3. eksctl
4. Helm
5. AWS SSM Plugin to enable using SSM on command line
6. SSM permissions added to the restricted admin role in your AWS account (RITM22003234427)

Setup:
1. Create VPC and subnets w/ Transit Gateway configured in the VPC routing table
2. Create your internal load balancer, and submit the request for the DNS entry that will point at the LB
3. Create your Akamai security groups, submitting AWS request to increase # of rules in each group if needed

## Cluster
1. Add your VPC and subnets to the bottlerocket.yml under the vpc section and edit cluster name.
2. Update the Akamai security group Ids in bottlerocket.yml under managedNodeGroups[0].securityGroups.attachIDs
3. Create cluster: ```eksctl create cluster -f bottlerocket.yml```

### Worldpay Internal DNS Configuration
It's important that we have our internal DNS configured to avoid the need for using hostAliases or IP addresses. 
Be sure not to add coredns as an 'addon' in the eksctl config yaml, or else the addon controller will periodically overwrite
the changes we make to the coredns ConfigMap.

Edit the configmap 'coredns' in the kube-system namespace to match what's in coredns-configmap.yml by adding the following section to the CoreFile:
```
    infoftps.com vantiv.com local fisglobal.com worldpay.com fnis.com {
      errors
      kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
      }
      cache 60
      forward . 10.10.10.10
    }
```
With the transit gateway added to the subnet routes, all IPs in 10.0.0.0/8 that aren't in local subnets will route
through the transit gateway into our on-prem network. However, some IPs like DB2 aren't in 10.0.0.0/8 and will need
their own entries in the routes, also pointing to the TGW.

It's also important that Java's indefinite DNS caching is disabled. Teams still using Dockerfiles should disable this
in the Dockerfile. Teams using Cloud Native Buildpacks (best practice) likely already have this covered (need to confirm
since the EKS cluster don't use nodelocaldns).

Non-EKS clusters running nodelocaldns have this config inside the nodelocaldns configmap instead.

### Artifactory Credentials
We can no longer use the Nexus image registry because it's not compatible with the container runtime that the BottleRocket AMI uses (containerd).
When checking to see if an image exists in the registry, containerd uses an HTTP HEAD request, and the response is usually either a 200 if already
authenticated and the image is present, or a 401 if authentication needs to happen. Instead, Nexus returns a 404 response, and it appears that the image does not exist in the registry,
even when does. This is a known issue with the version of Nexus that we're using: https://issues.sonatype.org/browse/NEXUS-12684.

We configure the Artifactory credentials through the control container that is provided by BottleRocket. 
Use AWS SSM to add the image registry credentials to the worker nodes all at once.

1. Set the base64 encoded username:password string inside of credentials.json
2. Replace the cluster name and nodegroup name to match the cluster and nodegroup you created in the command below
3. Run the command to execute the apiclient command inside each worker node that will set our Artifactory credentials, 
taking note of the commandId that is output:
```shell
aws ssm send-command --targets Key=tag:eks:nodegroup-name,Values=ng-1 \
--targets Key=tag:eks:cluster-name,Values=bottlerocket-eks-cluster \
--document-name "AWS-RunShellScript" \
--comment "Bottlerocket API Set Registry Credentials" \
--cli-input-json file://credentials.json \
--output text \
--query "Command.CommandId"
```
4. Run the following command to verify that the command was run succsesfully on each node:
```shell
aws ssm list-command-invocations --command-id "COMMANDID OUTPUT FROM PREVIOUS COMMAND" --details
```
## Update Your kubectl Config
The previous steps did update your kubectl config to point at the newly created cluster, but if you proceed you'll end up
with the following error when trying to run helm commands:

```Error: INSTALLATION FAILED: Kubernetes cluster unreachable: exec plugin: invalid apiVersion "client.authentication.k8s.io/v1alpha1"```

To fix this, just run:

```aws eks update-kubeconfig --name bottlerocket-eks-cluster```

## Prometheus and Grafana K8s Operators
1. Add the bitnami Helm repo: ```helm repo add bitnami https://charts.bitnami.com/bitnami```
2. Install Prometheus: ```helm install prometheus bitnami/kube-prometheus -f helm-configs/prometheus-config-bitnami.yml --namespace monitoring --create-namespace```
* It's a good idea to go into the EC2 Volumes and re-name the 100GB volume it provisions, so we know what it's for
* This installed before everything else, because a lot of our other helm charts will create a Prometheus ServiceMonitor for themselves
3. Install Grafana:
      1. Edit grafana-config-bitnami.yml so that the hostname in use matches the DNS entry that points to your load balancer
      2. TODO: Customize the config yml to create desired Ingress once load balancer and DNS are in place
      3. ```helm install grafana bitnami/grafana -f helm-configs/grafana-config-bitnami.yml --namespace monitoring```
* It's a good idea to go into the EC2 Volumes and re-name the 10GB volume it provisions, so we know what it's for

## K8s Dashboard
1. Add the Helm repo: ```helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/```
2. TODO: Customize the config yml to create desired Ingress once load balancer and DNS are in place
3. Install: ```helm install k8s-dashboard kubernetes-dashboard/kubernetes-dashboard --namespace dashboard --create-namespace -f helm-configs/k8s-dashboard-config.yml```

## Ingress Controller
1. Add the Ingress Controller Helm repo: ```helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx```
2. Install: ```helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace -f helm-configs/ingress-config.yml```

## Cluster Autoscaler
1. // TODO: Use IAM Roles for Service Accounts as a security best practice so every pod doesn't have the IAM role
2. Add Helm repo: ```helm repo add autoscaler https://kubernetes.github.io/autoscaler```
3. Update cluster name in cluster-autoscaler-config.yml
4. Install: ```helm install cluster-autoscaler autoscaler/cluster-autoscaler --namespace kube-system -f helm-configs/cluster-autoscaler-config.yml```

## GitHub Actions Self-Hosted Runner K8s Operator
1. Install cert-manager(required for actions-runner operator):
   1. Add Helm repo: ```helm repo add jetstack https://charts.jetstack.io```
   2. Install: ```helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true```
2. Install actions-runner-controller:
   1. Request a service account in GitHub that has access to your Organization
   2. Obtain/generate an access token from that service account
   3. Add the token to actions-runner-config.yml
   4. Add the actions-runner Helm repo: ```helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller```
   5. Install: ```helm install --namespace actions-runner-system --create-namespace --wait actions-runner-controller -f helm-configs/actions-runner-config.yml actions-runner-controller/actions-runner-controller```

For more information about how to create runners please see the documentation: \
https://github.com/actions-runner-controller/actions-runner-controller#usage

## Instana Agent
1. Add Helm repo: ```helm repo add instana https://agents.instana.io/helm```
2. Add the key to the instana-config.yml
3. Update the cluster name in instana-config.yml
4. Install: ```helm install instana-agent --namespace instana-agent --create-namespace instana/instana -f helm-configs/instana-config.yml```

## Service Accounts & RBAC

Service accounts are created in the default namespace. The deployer account will have roles bound to it inside each
namespace it needs to deploy to, even though it's located in the default namespace.

```shell
# deployment service account (not admin)
kubectl create serviceaccount deployer
# do this in each namespace your deployer needs to be able to deploy to
kubectl apply -f k8s-resources/deployer-role.yml -n [your namespace here]
kubectl create rolebinding deployer --role deployer --serviceaccount default:deployer -n [your namespace here]

# admin service account (for admins who want to use dashboard)
kubectl create serviceaccount admin
kubectl create clusterrolebinding admin --clusterrole cluster-admin --serviceaccount default:admin

# get tokens
kubectl get secrets
kubectl get secret -o jsonpath={.data.token} [secret name here] | base64 -d

# developer role in each namespace that will host developer apps
kubectl create serviceaccount developer -n [your namespace here]
kubectl apply -f k8s-resources/developer-role.yml -n [your namespace here]
kubectl create rolebinding developer --role developer --serviceaccount [your namespace here]:developer

# read-only role in each namespace that will host developer apps
kubectl create serviceaccount dashboard-read-only -n [your namespace here]
kubectl apply -f k8s-resources/read-only-role.yml -n [your namespace here]
kubectl create rolebinding dashboard-read-only --role dashboard-read-only --serviceaccount [your namespace here]:dashboard-read-only
```