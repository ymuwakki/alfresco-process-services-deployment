## Alfresco Process Services Deployment

The purpose of this project is to provide a deployment mechanism for Alfresco Process Services using Docker image via with/without Alfresco Identity Services and Kubernetes cluster using helm chart.

Check [prerequisites section](https://github.com/Alfresco/alfresco-dbp-deployment/blob/master/README-prerequisite.md) before you start.

### Kubernetes Cluster configuration:

You can choose to deploy the infrastructure to a local kubernetes cluster (illustrated using minikube) or you can choose to deploy to the cloud (illustrated using AWS).
Please check the Anaxes Shipyard documentation on [running a cluster](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/SECRETS.md).

Note the resource requirements:

Minikube: At least 12 gigs of memory, i.e.:

minikube start --memory 12000

AWS: A VPC and cluster with 5 nodes. Each node should be a m4.xlarge EC2 instance.

### OR

Example steps for running cluster in AWS: 

### AWS Kubernetes cluster configuration and s3 bucket:

##### 1. Create s3 bucket tos tore cluster configuration:
```bash
bucket_name=aps-k8s-cluster
export KOPS_NAME="aps.k8s.local"
export KOPS_STATE_STORE="s3://aps-k8s-cluster"

aws s3api create-bucket --bucket ${bucket_name} --region us-west-1 --create-bucket-configuration LocationConstraint=us-west-1
``` 
##### 2.Configuring kops and kubernetes-cli: 

Generate ssh key or use existing one to login cluster ec2 instances: 
```bash
ssh-keygen -t rsa -b 4096 -C "aps_public_key"
- brew install kops
- brew install kubernetes-cli
```

##### 3.Create Cluster:

```bash
kops create cluster --ssh-public-key ~/.ssh/k8s_cluster.pub --name=$KOPS_NAME --state=$KOPS_STATE_STORE --node-count 2 --zones=us-west-1a,us-west-1b --master-zones=us-west-1a,us-west-1b,us-west-1c --cloud=aws --node-size m4.xlarge --master-size t2.medium -v 10 --kubernetes-version "1.9.7" --topology private --networking weave --yes
```

### OR

##### If you have your own DNS name:
Set DOMAIN to the DNS Zone you used when [creating the cluster](https://github.com/kubernetes/kops/blob/master/docs/aws.md#scenario-1b-a-subdomain-under-a-domain-purchasedhosted-via-aws)

```bash
kops create cluster --ssh-public-key ~/.ssh/k8s_cluster.pub --name=$KOPS_NAME --state=$KOPS_STATE_STORE --node-count 2 --zones=us-west-1a,us-west-1b --master-zones=us-west-1a,us-west-1b,us-west-1c --cloud=aws --node-size m4.xlarge --master-size t2.medium -v 10 --kubernetes-version "1.9.7" --dns-zone=envalfresco.com --topology private --networking weave --yes
```

Wait for nodes to join the cluster and run the validate cluster before running next commands. 

```bash
kops validate cluster
kubectl cluster-info
```
Make note of your kubernetes dashboard URI 
Example : https://api-aps-k8s-local-e95f94-618045732.us-west-1.elb.amazonaws.com/ui

##### 4.Deploy kops dashboard:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
Kubernetes username: admin
```bash
Get admin password:
kops get secrets kube --type secret -oplaintext

Token:
kops get secrets admin --type secret -oplaintext
```

kubectl config current-context - To make sure we are pointed to the correct clustername

##### 5. Helm Tiller
Initialize the Helm Tiller:

```bash
helm init --upgrade
helm repo update
helm version
```

##### 6. Create Namespace: 
```bash
export DESIREDNAMESPACE=aps-deploy
kubectl create namespace $DESIREDNAMESPACE

kubectl create serviceaccount tiller --namespace $DESIREDNAMESPACE

helm init --service-account tiller

kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin \
--user=admin --user=kubelet --group=system:serviceaccounts
```
###################################################################################

#### Deploy Alfresco process services using helm chart in existing cluster:

##### 1.Install the nginx-ingress-controller

Install the nginx-ingress-controller into your cluster
```bash
helm repo update
helm install stable/nginx-ingress --version 0.8.11 --set controller.scope.enabled=true \
--set controller.scope.namespace=$DESIREDNAMESPACE --namespace $DESIREDNAMESPACE
```

##### 2.Get the nginx-ingress-controller release name from the previous command and set it as a variable

(nginx-ingress-controller release name)

```bash
helm list
export INGRESSRELEASE=<name>
export INGRESSRELEASE=vested-dragonfly
helm status $INGRESSRELEASE
```

##### 3.Get the nginx-ingress-controller port for the infrastructure and Get Minikube or ELB IP and set it as a variable for future use:

ON MINIKUBE:
```bash
export INFRAPORT=$(kubectl get service $INGRESSRELEASE-nginx-ingress-controller \
--namespace $DESIREDNAMESPACE -o jsonpath={.spec.ports[0].nodePort})
export ELBADDRESS=$(minikube ip)
```
ON AWS: 
```bash
export ELBADDRESS=$(kubectl get services $INGRESSRELEASE-nginx-ingress-controller \
--namespace=$DESIREDNAMESPACE -o jsonpath={.status.loadBalancer.ingress[0].hostname})
```

##### 4.EFS Storage (NOTE! ONLY FOR AWS!)
Create a EFS storage on AWS and make sure it is in the same VPC as your cluster. Make sure you open inbound traffic in the security group to allow NFS traffic. Save the name of the server ex:

fs-6f1af576.efs.us-west-1.amazonaws.com

Imp note: open the port for 'NFS' on default security group of VPC (i.e default - security group name) to '0.0.0.0/0 or your own subnet' 
```bash
export NFSSERVER=<dnsnameforEFS>
example: export NFSSERVER=fs-6f1af576.efs.us-west-1.amazonaws.com
```
Note! The Persistent volume created to store the data on the created EFS has the ReclaimPolicy set to Recycle. This means that by default, when you delete the release the saved data is deleted automatically.

To change this behaviour and keep the data you can set the alfresco-infrastructure.persistence.reclaimPolicy value to Retain. For more Information on Reclaim Policies checkout the official K8S documentation [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaim-policy)

We don't advise you to use the same EFS instance for persisting the data from multiple dbp deployments.

##### 5.Add the Alfresco Kubernetes repository to helm.
helm repo add alfresco-incubator https://kubernetes-charts.alfresco.com/incubator

##### 6.Optional! Adding an Alfresco Process Services license at deploytime

Create a secret from your activiti.lic file.
```bash
kubectl create secret generic licenseaps --from-file=./activiti.lic --namespace=$DESIREDNAMESPACE
```

##### 7.a.Deploy Alfresco Process Services (default option):

```bash
#On MINIKUBE
helm install alfresco-incubator/alfresco-process-services --set dnsaddress="http://$ELBADDRESS:$INFRAPORT" \
--namespace=$DESIREDNAMESPACE --set license.secretName=licenseaps

#On AWS
### without Identity services:
helm install alfresco-incubator/alfresco-process-services --set dnsaddress="http://$ELBADDRESS" \
--namespace=$DESIREDNAMESPACE --set license.secretName=licenseaps
```

### OR

##### 7.b.APS deployment using Identity Service authentication:

Important (Only for AWS): Enabling Identity Services and using valid DNS name, you need self signed or valid registered SSL certificate for DNS name.

Note: Documentation link attached in Documentation Guide section to understand DNS, SSL, ELB and AWS certificate Manager

Please find more information [Using Alfresco Identity service]:(https://github.com/Alfresco/alfresco-identity-service)

Find the ELB for of the nodes which need to be added to 'A-NAME/C-NAME' record set on Route53 and upload SSL certificate to the ELB

```bash
echo $ELBADDRESS
```

##### Create a Route 53 Record Set in your Hosted Zone

* Go to AWS Management Console and open the Route 53 console.
* Click Hosted Zones in the left navigation panel, then Create Record Set.
* In the Name field, enter your "$ELBADDRESS" defined in step 3.
* In the Alias Target, select your ELB address ("$ELBADDRESS").
* Click Create.

```bash
export DNSNAME="apsk8s.envalfresco.com"

helm install alfresco-incubator/alfresco-process-services --namespace=$DESIREDNAMESPACE \
--set alfresco-infrastructure.persistence.efs.dns="$NFSSERVER" \
--set dnsaddress="http://$DNSNAME",license.secretName=licenseaps,activiti_env.IDENTITY_SERVICE_AUTH=https://"$DNSNAME/auth",alfresco-infrastructure.alfresco-identity-service.enabled=true,activiti_env.IDENTITY_SERVICE_ENABLED=true \
--set alfresco-infrastructure.alfresco-identity-service.client.alfresco.redirectUris=['\"'http://$DNSNAME*'"\']
```

##### Output:
Once deployed we will get APS end points like below output including kubernetes services details

"You can access Alfresco Process Services using this address:

  Alfresco Process Services Activiti App: http://ELBADDRESS/activiti-app or http://DNSADDRESS/activiti-app 

  Alfresco Process Services Admin App: http://ELBADDRESS/activiti-admin or http://DNSADDRESS/activiti-admin"

Below values used for APS login by default:

  | Property                      | Value                    |
  | ----------------------------- | ------------------------ |
  | Admin User Username           | `admin`                  |
  | Admin User Password           | `admin`                  |
  | Admin User Email              | `admin@app.activiti.com` |
  | Alfresco Client Redirect URIs | `http://localhost*`      |
  
##### 8.Teardown and Useful helm commands:

```bash
helm delete $INGRESSRELEASE

helm delete $APSRELEASE

kubectl delete namespace $DESIREDNAMESPACE

kops delete cluster <clustername>

Note: Delete EFS volume manually before deleting cluster. 

helm upgarde <releasename> <chartpath> extra variables confir

helm rollback <releasename> <revisionnumber>

helm delete <releasename>
```

Depending on your cluster type you should be able to also delete it if you want. For minikube you can just run to delete the whole minikube vm.
```bash
minikube delete
```

##### Documentation Guide: 
For more information on running and tearing down k8s environments, follow this [guide](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/docs/running-a-cluster.md).

To know [Using Alfresco Identity service](https://github.com/Alfresco/alfresco-identity-service).

Documents to understand SSL certificate, ELB and AWS certificate manager:
[Getting public or private SSL certificate in AWS](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html). 

Importing Certificates into AWS Certificate Manager. [Importing certificate to AWS certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/import-certificate.html).

Attach SSL Certificates to AWS ELB [Add Certificates to your existing ELB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-certificates.html).

