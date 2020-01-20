# aks-applicationgateway-ingresscontroller

AKS Architecture - Azure Application Gateway Ingress Controller

The Application Gateway Ingress Controller allows the Azure Application Gateway to be used as the ingress for an Azure Kubernetes Service aka AKS cluster..

As shown in the figure below, the ingress controller runs as a pod within the AKS cluster. It consumes Kubernetes Ingress Resources and converts them to an Azure Application Gateway configuration which allows the gateway to load-balance traffic to Kubernetes pods.

 

Step 1: 
Connecting terminal...
hugo@Azure:~$ az account list --output table
Name                                                 CloudName    SubscriptionId                        State    IsDefault
---------------------------------------------------  -----------  ------------------------------------  -------  -----------
Visual Studio Enterprise                             AzureCloud   aa5c3570-71cb-42ad-8b68-xxxxxxxxx  Enabled  False

Step 2: 
hugo@Azure:~$ az account set --subscription aa5c3570-71cb-42ad-8b68-xxxxxxxxxx

Step 3:
hugo@Azure:~$ az ad sp create-for-rbac --skip-assignment
{
  "appId": "6414b10e-fd04-49ec-9068- xxxxxxxxxxxx ",
  "displayName": "azure-cli-2019-06-04-13-11-51",
  "name": "http://azure-cli-2019-06-04-13-11-51",
  "password": "e349913d-9567-xxxxxxxxxxxx",
  "tenant": "72f988bf-86f1-41af-91ab- xxxxxxxxxxxx "
}
hugo@Azure:~$

Step 4:
hugo@Azure:~$ az ad sp show --id <appId> --query "objectId"
hugo@Azure:~$ az ad sp show --id 6414b10e-fd04-49ec-9068- xxxxxxxxxxxx --query "objectId"
"3fb7de6c-fb4a-4de1-a887- xxxxxxxxxxxx "

Step 5:
After creating the service principal in the step above, click to create a custom template deployment. Provide the appId for servicePrincipalClientId, password and objectId in the parameters. Note: For deploying an RBAC enabled cluster, set aksEnabledRBAC parameter to true. (I used false in my deployment)
Azure Template: 
https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2Fapplication-gateway-kubernetes-ingress%2Fmaster%2Fdeploy%2Fazuredeploy.json

 


The templated deployment will create the following:
Managed Identity
Virtual Network
Public IP Address
Application Gateway
Azure Kubernetes Service

 

Step 6:
After the deployment completes, you will find the parameters needed for the steps below in the deployment outputs window. 
(Navigate to the deployment's output by following this path in the Azure portal: Home > resource group > Deployments > new deployment > Outputs)

 


Setting up Application Gateway Ingress Controller on AKS
With the instructions in the previous section we created and configured a new Azure Kubernetes Service (AKS) cluster. We are now ready to deploy to our new Kubernetes infrastructure. The instructions below will guide us through the process of installing the following 2 components on our new AKS:
•	Azure Active Directory Pod Identity - Provides token-based access to the Azure Resource Manager (ARM) via user-assigned identity. Adding this system will result in the installation of the following within your AKS cluster:
o	Custom Kubernetes resource definitions: AzureIdentity, AzureAssignedIdentity, AzureIdentityBinding
o	Managed Identity Controller (MIC) component
o	Node Managed Identity (NMI) component
•	Application Gateway Ingress Controller - This is the controller which monitors ingress-related events and actively keeps your Azure Application Gateway installation in sync with the changes within the AKS cluster.

Step 7:
To configure kubectl to connect to the deployed Azure Kubernetes Cluster:
az aks get-credentials --resource-group <ResourceGroup> --name <AKSCluster>
(you can find booth information on the Resource Group that you created on the previous step:

 

hugo@Azure:~$ az aks get-credentials --resource-group AKS-APPGW-IC --name aks1a7f
Merged "aks1a7f" as current context in /home/hugo/.kube/config

Step 8:
hugo@Azure:~$ kubectl get nodes
NAME                       STATUS   ROLES   AGE   VERSION
aks-agentpool-10356395-0   Ready    agent   48m   v1.12.7
aks-agentpool-10356395-1   Ready    agent   48m   v1.12.7
aks-agentpool-10356395-2   Ready    agent   48m   v1.12.7
hugo@Azure:~$

Step 9:
Add aad pod identity service to the cluster using the following command. This service will be used by the ingress controller.
Attention, in my scenario, I have disable RBAC, so I will use this:
hugo@Azure:~$ kubectl create -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment.yaml 
customresourcedefinition.apiextensions.k8s.io/azureassignedidentities.aadpodidentity.k8s.io created
customresourcedefinition.apiextensions.k8s.io/azureidentitybindings.aadpodidentity.k8s.io created
customresourcedefinition.apiextensions.k8s.io/azureidentities.aadpodidentity.k8s.io created
daemonset.extensions/nmi created
deployment.extensions/mic created

In case of enable RBAC, please use this: (NOT USED in MY Environment): 
kubectl create -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml 

Step 10:
Install Helm and run the following to add application-gateway-kubernetes-ingress helm package:
Attention, in my scenario, I have disable RBAC, so I will use this:
hugo@Azure:~$ helm init
$HELM_HOME has been configured at /home/hugo/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
hugo@Azure:~$ helm repo add application-gateway-kubernetes-ingress https://azure.github.io/application-gateway-kubernetes-ingress/helm/
"application-gateway-kubernetes-ingress" has been added to your repositories
hugo@Azure:~$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "application-gateway-kubernetes-ingress" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.


In case of enable RBAC, please use this: (NOT USED in MY Environment):
kubectl create serviceaccount --namespace kube-system tiller-sa
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller-sa
helm init --tiller-namespace kube-system --service-account tiller-sa
helm repo add application-gateway-kubernetes-ingress https://azure.github.io/application-gateway-kubernetes-ingress/helm/
helm repo update

Step 11:
Edit helm-config.yaml and fill in the values for appgw and armAuth and aksClusterConfiguration:
You can find the data at the step 6

hugo@Azure:~$ vi helm-config.yaml


# This file contains the essential configs for the ingress controller helm chart

################################################################################
# Specify which application gateway the ingress controller will manage
#
appgw:
    subscriptionId: aa5c3570-71cb-42ad-8b68- xxxxxxxxxxxx
    resourceGroup: AKS-APPGW-IC
    name: AKS-APPGW-IC

################################################################################
# Specify which kubernetes namespace the ingress controller will watch
# Default value is "default"
#
# kubernetes:
#   watchNamespace: <namespace>

################################################################################
# Specify the authentication with Azure Resource Manager
#
# Two authentication methods are available:
# - Option 1: AAD-Pod-Identity (https://github.com/Azure/aad-pod-identity)
armAuth:
    type: aadPodIdentity
    identityResourceID: /subscriptions/aa5c3570-71cb-42ad-8b68- xxxxxxxxxxxx /resourceGroups/AKS-APPGW-IC/providers/Microsoft.ManagedIdentity/userAssignedIdentities/appgwContrIdentity9ed1
    identityClientID:  dec7f9af-9012-4cd5-8ca0- xxxxxxxxxxxx

################################################################################
# Specify if the cluster is RBAC enabled or not
rbac:
    enabled: false # true/false

################################################################################
# Specify aks cluster related information
aksClusterConfiguration:
    apiServerAddress: aksappgwic001-d97d34ea.hcp.eastus2.azmk8s.io

Step 12:
Then execute the following to the install the Application Gateway ingress controller package.
hugo@Azure:~$ helm install -f helm-config.yaml application-gateway-kubernetes-ingress/ingress-azure
NAME:   fair-skunk
LAST DEPLOYED: Tue Jun  4 15:01:05 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/AzureIdentity
NAME                           AGE
fair-skunk-azid-ingress-azure  0s

==> v1/AzureIdentityBinding
NAME                                  AGE
fair-skunk-azidbinding-ingress-azure  0s

==> v1/ConfigMap
NAME                         DATA  AGE
fair-skunk-cm-ingress-azure  4     0s

==> v1/ServiceAccount
NAME                         SECRETS  AGE
fair-skunk-sa-ingress-azure  1        0s

==> v1beta2/Deployment
NAME                      READY  UP-TO-DATE  AVAILABLE  AGE
fair-skunk-ingress-azure  0/1    0           0          0s


NOTES:
Thank you for installing ingress-azure:0.5.0.

Your release is named fair-skunk.
The controller is deployed in deployment fair-skunk-ingress-azure.

Configuration Details:
----------------------
* AzureRM Authentication Method:
    - Use AAD-Pod-Identity
* Application Gateway:
    - Subscription ID : aa5c3570-71cb-42ad-8b68-7c4457967950
    - Resource Group  : AKS-APPGW-IC
    - Application Gateway Name : AKS-APPGW-IC
* Kubernetes:
    - Watch Namespace : default

Please make sure the associated aadpodidentity and aadpodidbinding is configured.
For more information on AAD-Pod-Identity, please visit https://github.com/Azure/aad-pod-identity

Expose the AKS service over HTTP or HTTPS, to the internet, using an Azure Application Gateway that you deployed.
Deploy guestbook application
The guestbook application is a canonical Kubernetes application that composes of a Web UI frontend, a backend and a Redis database. By default, guestbook exposes its application through a service with name frontend on port 80. Without a Kubernetes Ingress Resource the service is not accessible from outside the AKS cluster. We will use the application and setup Ingress Resources to access the application through HTTP and HTTPS.
Follow the instructions below to deploy the guestbook application.

Step 13: 
Download guestbook-all-in-one.yaml from this link: 
https://raw.githubusercontent.com/kubernetes/examples/master/guestbook/all-in-one/guestbook-all-in-one.yaml 

Step 14:
hugo@Azure:~$ vi guestbook-all-in-one.yaml
(copy the app in this file)

Step 15:
Deploy guestbook-all-in-one.yaml into your AKS cluster by running:
hugo@Azure:~$ kubectl apply -f guestbook-all-in-one.yaml
service/redis-master created
deployment.apps/redis-master created
service/redis-slave created
deployment.apps/redis-slave created
service/frontend created
deployment.apps/frontend created
hugo@Azure:~$

Expose services over HTTP
In order to expose the guestbook application we will using the following ingress resource:

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: guestbook
 annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: frontend
          servicePort: 80

This ingress will expose the frontend service of the guestbook-all-in-one deployment as a default backend of the Application Gateway.

Step 16:
Save the above ingress resource as ing-guestbook.yaml.
hugo@Azure:~$ vi ing-guestbook.yaml

Step 17:
Deploy ing-guestbook.yaml by running:
hugo@Azure:~$ kubectl apply -f ing-guestbook.yaml
ingress.extensions/guestbook created
At this time, you can check that we have our application up and running through the Application Gateway performing as an ingress controller.
Please, check the step below named Verification, to see the application.

Step 18:
Deploy an Azure Front Door.
Considering an extra layer of protection, we will install the Azure Front Door to integrate with the Azure Application Gateway. (We will have 1 backend only for the demonstration proposal, but you can add as much backends as you need to accommodate your customer requirements.
The ip address of the Application Gateway was used as the mentioned backend pool.

 


Verification:
1.	Access the Front End IP Address to see the application up and running: http://52.179.224.169/ 
2.	Please check that the Resources created on the Resource Group has only one public ip address:
 
3.	Check the on Application Gateway, the Backend 

 
4.	Now, finally, check the application running considering the Archoitecture highlighted on the introduction:
a.	Azure Front Door > Application Gateway > AKS > Container > Application 
b.	Access here: http://aksappgwic.azurefd.net/ (view from the user seat at the internet)
 

Final view of items deployed:
 


References: 
https://github.com/Azure/application-gateway-kubernetes-ingress/blob/master/docs/tutorial.md 
