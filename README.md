# Google-Anthos-setup-configuration

# Anthos In-cluster Internal HTTPS ASM Gateway Setup

## Prerequisites

1. A GKE cluster with ASM Supported GKE Version, here I'm using 1.22.8-gke.201

2. As we're going tob create a private cluster so be sure you've below priviledges assosciated with Bastion VM Service Account

- Kubernetes Engine Admin
- GKE Hub Admin
- Mesh Config Admin
- Project IAM Admin
- Service Usage Admin
- Compute Instance Admin (v1)
& some monitoring roles(If required)

3. API required - 

- Anthos API
- GKE hub API
- Mesh Configuration API
- Mesh CA API
- Stackdriver API
& some monitoring APIs(If required)

To enable the  APIs you need a security admin role or any Basic high-level role like owner or editor.
```
gcloud services enable \
    container.googleapis.com \
    compute.googleapis.com \
    monitoring.googleapis.com \
    logging.googleapis.com \
    cloudtrace.googleapis.com \
    meshca.googleapis.com \
    meshtelemetry.googleapis.com \
    meshconfig.googleapis.com \
    iamcredentials.googleapis.com \
    anthos.googleapis.com \
    gkeconnect.googleapis.com \
    gkehub.googleapis.com \
    cloudresourcemanager.googleapis.com \
    mesh.googleapis.com
```

Reference Link - https://cloud.google.com/service-mesh/v1.4/docs/gke-install-new-cluster

4. If you don't have, please follow the below repo to create a private cluster

https://gitlab.com/abhishek7389/hdfc-clo-prod/-/tree/main/env/prod/regional_resources/asia-south1/gke_cluster

Note - You need a cluster with at least 4vCPU Machine type(ex. e2-standard-4) with 2 nodes to install & connect it with Anthos Mesh Service. Also if you’re working in a shared vpc environment be sure you’ll have container.googleapis.com API enabled & a default GKE service account as a subnet user in a specified subnet. Also it private google access should be on state on subnet level.


## ASM installation

1. Seeting up firewall rule for your cluster 

1.1 Finding cluster master firewall 
```
gcloud compute firewall-rules list --filter="name~gke-CLUSTER_NAME-[0-9a-z]*-master" --project <network project id> 
``` 
1.2 Modify the firewall-rule with the required ports 
```
gcloud compute firewall-rules update FIREWALL_RULE_NAME --allow tcp:10250,tcp:443,tcp:15017,tcp:15014,tcp:8080,tcp:15021 --project <network project id>   
```
Note - Be sure your bastion is in cluster's control plane authorizied network

2. Authenticate your cluster in the terminal 
```
gcloud container clusters get-credentials hdfc-test-new-cluster --region asia-south1 --project searce-playground-v1
```
3. Getting your network ID, If you're working with shared network then it is required parameter
```
gcloud compute networks describe <your network name> --project <network project id> | grep id

```
4. Installation of ASM CLI 

4.1 Download the ASM CLI 
```
curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.12 > asmcli
```
4.2 Make the CLI Executable
```
chmod +x asmcli
```
5. Dry run the In-cluster ASM Installation
```
./asmcli validate \
  --project_id <PROJECT_ID> \
  --cluster_name <CLUSTER_NAME> \
  --cluster_location <CLUSTER_ZONE> \
  --fleet_id <PROJECT_ID> \
  --output_dir ./asm_output
```
Note - Please ignore warnings 

6. Installation of ASM 
```
./asmcli install \
  --project_id <PROJECT_ID> \
  --cluster_name $CLUSTER_NAME \
  --cluster_location <CLUSTER_NAME> \
  --fleet_id <PROJECT_ID> \
  --network_id "<network id>" \
  --output_dir ./asm_output \
  --enable_all \
  --option legacy-default-ingressgateway \
  --ca mesh_ca \
  --enable_gcp_components
```
Note - Flag --option legacy-default-ingressgateway create a ingress gateway for you. If you don't want default gateway then just remove this flag.

7. Get the Revision Installed & Lebel all the application namespaces so that every pod in the namespace sends metrics to the anthos dashboard

7.1 Getting the ASM revision installed
```
REVISION=$(kubectl get deploy -n istio-system -l app=istiod -o \
jsonpath={.items[*].metadata.labels.'istio\.io\/rev'}'{"\n"}')
```
7.2 Lebal the namespaces
```
kubectl label namespace <namespace name> istio-injection=enabled istio.io/rev=$REVISION --overwrite
```

example - kubectl label namespace istio-system istio-injection=enabled istio.io/rev=$REVISION --overwrite

## Deployment of ingress controller 

create a seperate namespace for the gateway
```
kubectl create ns istio-gateway
```
label the namespace
```
kubectl label namespace istio-gateway istio-injection=enabled istio.io/rev=$REVISION --overwrite
```

1. Create a self managed certificate & kubernetes TLS secret

2.1 Please follow the doc to create a certificarte - https://docs.google.com/document/d/1RqTc1JelJNfAX2xQ2WqQtrIe8QjyNDGwEOyNTNwRT9E/edit

2.2 Create a TLS secret in the kubernetes 
```
kubectl create -n istio-gateway secret tls asm-gateway-credential --key=asm.key --cert=asm.crt
```


2. Deploy the files one by one 

- seriveaccount.yaml
- role.yaml
- deployment.yaml
- service.yaml

Note - Be sure, you've labelled the namespace with istio-injection

## Deployment of Application 

Create a separate namespace for the application
```
kubectl create ns ns3
```
label the namespace
```
kubectl label namespace ns3 istio-injection- istio.io/rev=$REVISION --overwrite
```
1. Clone the application yamls
```
kubectl apply -f <yaml name>
```
Note - Be sure, you've labelled the namespace with istio-injection

2. Create a Gateway 

please apply the below yaml, in application namespace

- gateway.yaml

3. Create a virtual service to map application service

- virtual-servie.yaml

### Note: If your Gateway is in different namespace & you want to map your application's virtual service from different namespace be sure you reference your gateway like mention below - 
```
gateways:
  - <Your Gateway's Namespace>/<Your Gateway Name>
```
example: in our case gateway is deployed in application namespace(ie ns3) so if there are requirement thatb you need to reference this gateway from the different namespace virtual service file so in the virtual file you will reference the gateway section mentioned above like -
```
gateway:
 - ns3/asm-gateway
```

## Testing the application

1. Map your ingress controller service IP with your certificate hostname in the hosts file
```
sudo vi /etc/hosts
```
add the below content

<ingress controller service IP> <your hostname>

2. test by requesting the URL
```
curl -iv --cacert <path to your cert> https://<hostname>
```


Reference Links - 

- ASM Firewall Rules - https://cloud.google.com/service-mesh/docs/private-cluster-open-port
- Hands on lab - https://www.cloudskillsboost.google/focuses/8459
- Self managed certificate - https://cloud.google.com/load-balancing/docs/ssl-certificates/self-managed-certs#gcloud
- Istio Docs for HTTPS - https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/

