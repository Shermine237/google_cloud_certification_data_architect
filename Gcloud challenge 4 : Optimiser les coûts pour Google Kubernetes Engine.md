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
```
