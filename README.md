# MCI

### Setup up Multi Cluster Ingress
Before moving foreward, let's set PROJECT_ID variable 
```bash
export PROJECT_ID=<<your_Project_ID>>
```

##### Enable APIs
We are using standalone MultiClusterIngress Pricing, so will make sure Anthos APIs is disbaled, if you are using other Anthos features then no need to disable anthos API.

```bash
# Confirm Anthos API is disabled 
gcloud services list --project=${PROJECT_ID} | grep anthos.googleapis.com

# Enable required APIs
gcloud services enable \
    multiclusteringress.googleapis.com \
    gkehub.googleapis.com \
    container.googleapis.com \
    multiclusterservicediscovery.googleapis.com \
    --project=${PROJECT_ID}
```
</br>

##### Deploy clusters
Create two GKE clusters named gke-us and gke-eu in the europe-west1 and us-central1 zones, with Workload Identity enabled.

```bash
# Create the gke-us cluster in the us-central1-b zone:
gcloud container clusters create gke-us \
    --zone=us-central1-b \
    --enable-ip-alias \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --release-channel=stable \
    --project=${PROJECT_ID}

# Create the gke-eu cluster in the europe-west1-b zone:
gcloud container clusters create gke-eu \
    --zone=europe-west1-b \
    --enable-ip-alias \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --release-channel=stable \
    --project=${PROJECT_ID}
```
</br>

##### Configure Cluster Credentials

```bash
gcloud container clusters get-credentials gke-us \
    --zone=us-central1-b \
    --project=${PROJECT_ID}

gcloud container clusters get-credentials gke-eu \
    --zone=europe-west1-b \
    --project=${PROJECT_ID}
```
</br>

##### Rename Cluster Contexts

```bash
kubectl config rename-context gke_${PROJECT_ID}_us-central1-b_gke-us gke-us
kubectl config rename-context gke_${PROJECT_ID}_europe-west1-b_gke-eu gke-eu
```
</br>

##### Register Clusters to fleet

```bash
# Register the clusters
gcloud container fleet memberships register gke-us \
    --gke-cluster us-central1-b/gke-us \
    --enable-workload-identity \
    --project=${PROJECT_ID}

gcloud container fleet memberships register gke-eu \
    --gke-cluster europe-west1-b/gke-eu \
    --enable-workload-identity \
    --project=${PROJECT_ID}
    
# Confirm clusters are part of fleet
gcloud container fleet memberships list --project=${PROJECT_ID}
```
</br>

##### Specify a config cluster

```bash
# Enable Multi Cluster Ingress and select gke-us as the config cluster:
gcloud container fleet ingress enable \
    --config-membership=gke-us \
    --project=${PROJECT_ID}
#The config cluster takes up to 15 minutes to register. 
```
</br>

### Deploying Ingress accross Clusters

##### Creating Namespace
Because fleets have the property of namespace sameness, we recommend that you coordinate Namespace creation and management across clusters so identical Namespaces are owned and managed by the same group

```bash
# Switch to the gke-us context and create namespace
kubectl config use-context gke-us
kubectl apply -f namespace.yaml

# Switch to the gke-us context and create namespace
kubectl config use-context gke-eu
kubectl apply -f namespace.yaml
```
</br>

##### Deploying App

```bash
# Switch to the gke-us context and deploy app
kubectl config use-context gke-us
kubectl apply -f deployment.yaml

# Switch to the gke-us context and deploy app
kubectl config use-context gke-eu
kubectl apply -f deployment.yaml
```
</br>

##### Deploying MCS through Config Cluster
Now that the application is deployed across gke-us and gke-eu, you will deploy a load balancer by deploying MultiClusterIngress and MultiClusterService resources in the config cluster. These are the multi-cluster equivalents of Ingress and Service resources.

```bash
# Switch to the gke-us context
kubectl config use-context gke-us

# Deploy the MultiClusterService
kubectl apply -f mcs.yaml

#This MultiClusterService creates a derived headless Service in every cluster that matches Pods with app: global in global namespace

```
</br>

##### Deploying MCI through Config Cluster
MultiClusterIngress needs to be deployed in Config Cluster

```bash
# Switch to the gke-us context
kubectl config use-context gke-us

# Deploy the MultiClusterIngress
kubectl apply -f mci.yaml
```
</br>

##### Verify Deployment Status

```bash
# Switch to the gke-us context
kubectl config use-context gke-us

# Verify that deployment has succeeded:
kubectl describe mci global-ingress -n global

#Validate that the application is serving on the VIP with the /ping endpoint:
curl INGRESS_VIP/ping
```
</br>


