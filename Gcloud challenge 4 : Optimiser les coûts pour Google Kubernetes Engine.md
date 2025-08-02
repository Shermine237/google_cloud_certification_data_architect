# Optimise Costs for Google Kubernetes Engine: Google Cloud Challenge Lab Walkthrough

## Initial Config
```bash
gcloud auth list

PRJ=$DEVSHELL_PROJECT_ID
ZONE=<ENTER ZONE>
CLUSTER=<ENTER CLUSTER NAME>
MACH_TYPE=e2-standard-2
REL_CHANNEL=rapid
ENV_DEV=dev
ENV_PRD=prod

# Set the zone, so we don't have to keep specifying in our kubectl commands
gcloud config set compute/zone ${ZONE}
```

## Task 1 — Create a Cluster and Deploy Your App
### Create cluster
```bash
# Create the zonal cluster - this will take a few minutes
gcloud container clusters create ${CLUSTER} \
  --num-nodes=2 \
  --machine-type=${MACH_TYPE} \
  --release-channel=${REL_CHANNEL} \
  --enable-vertical-pod-autoscaling # just in case we need it!
```

### Now we need to create two namespaces, for the dev and prod environments
```bash
kubectl create namespace ${ENV_DEV}
kubectl create namespace ${ENV_PRD}
```

### Deploy application
> We’re given the command to download the OnlineBoutique application, and to deploy it to the dev namespace. Here we apply a pre-created manifest.
```bash
# Clone the application
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git

# Deploy to the Dev namespace
cd microservices-demo
kubectl apply -f ./release/kubernetes-manifests.yaml --namespace dev
```

> Verify :
```bash
# Look at the nodes
kubectl get nodes

# Look at the namespaces
kubectl get namespace
```
> set our namespace to DEV, so we don't have to specify with each kubectl command
```bash
kubectl config set-context --current --namespace=${ENV_DEV}
```

> Verify
```bash
# Look at the pods
kubectl get pods

# inspect the deployment
kubectl get deployment
```

## Task 2 — Migrate to an Optimised Node Pool
### Config
```bash
DEFAULT_POOL=default-pool
NEW_POOL=<ENTER POOL NAME>
CUSTOM_TYPE=custom-2-3584
```

### Create new node pool
```bash
# Create new node pool with e2-standard-2
gcloud container node-pools create ${NEW_POOL} \
  --cluster=${CLUSTER} \
  --machine-type=${CUSTOM_TYPE} \
  --num-nodes=2
```

### Migrate our application
> Now we’re going to migrate our application to the new node pool. To do this, we have to cordon off the default-pool node pool, drain it, and migrate our workloads to the new node pool
```bash
# Cordon off the existing node pool, to stop new nodes being scheduled to it
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=${DEFAULT_POOL} -o=name); do
  kubectl cordon "$node";
done

# Drain the existing node pool
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=${DEFAULT_POOL} -o=name); do
  kubectl drain --force --ignore-daemonsets --delete-emptydir-data --grace-period=10 "$node";
done
```

### Delete the old node pool
```bash
# Delete the old node pool
gcloud container node-pools delete ${DEFAULT_POOL} \
  --cluster ${CLUSTER}
```

## Task 3 — Apply a Frontend Update
### Set a pod disruption budget,
```bash
PDB=onlineboutique-frontend-pdb

kubectl create poddisruptionbudget ${PDB} \
  --selector=run=frontend --min-available 1
```

### Edit manifest
> We’re told to edit our frontend to point to:
```bash
gcr.io/qwiklabs-resources/onlineboutique-frontend:v2.1
```
> Let’s edit our file ./release/kubernetes-manifests.yaml and update our image. We can do this from the Cloud Shell Editor:
<img width="1400" height="887" alt="image" src="https://github.com/user-attachments/assets/36b0a7e9-0905-4347-95b5-0b459dcfb5cf" />

> Whilst you’re there, don’t forget to change the ImagePullPolicy as requested in the instructions. (This forces the deployment to always refresh from the registry.)
<img width="997" height="502" alt="image" src="https://github.com/user-attachments/assets/3970ed46-05cc-4daf-a05b-62085df5ccc3" />

### Apply
> Now we can redeploy the manifests:
```bash
kubectl apply -f ./release/kubernetes-manifests.yaml --namespace dev
```

## Task 4 — Autoscale from Estimated Traffic
### Setup the HPA, to maintain 50% CPU across all pods
> We need to plan for a large spike in traffic. We’re told to setup the horizontal pod autoscaler (HPA) as follows:

    CPU target of 50%
    Between 1 and <specified> number of replicas.
```bash
MAX_REPLICAS=<ENTER VALUE>

# Setup the HPA, to maintain 50% CPU across all pods
kubectl autoscale deployment frontend \
  --cpu-percent=50 --min=1 --max=${MAX_REPLICAS}

# check HPA status
kubectl get hpa
```

### Enable cluster (node) autoscaling
> We also need to configure the cluster autoscaler, so that we can provision and destroy nodes as necessary, to accommodate our autoscaling pods. We’re told to set up the cluster autoscaler to scale between 1 and 6 nodes, inclusive.
```bash
# Enable cluster (node) autoscaling
gcloud beta container clusters update ${CLUSTER} \
  --enable-autoscaling --min-nodes 1 --max-nodes 6
```

## Load test with Locust
> Here, we’re given a command to load test with Locust:
```bash
FRONTEND_EXTERNAL_IP=<enter IP>
kubectl exec $(kubectl get pod --namespace=dev | grep 'loadgenerator' | cut -f1 -d ' ') -it --namespace=dev -- bash -c 'export USERS=8000; locust --host="http://${FRONTEND_EXTERNAL_IP}" --headless -u "8000" 2>&1'
```

# And that’s all we have to do! Lab done!
