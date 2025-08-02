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
```bash
# Create the zonal cluster - this will take a few minutes
gcloud container clusters create ${CLUSTER} \
  --num-nodes=2 \
  --machine-type=${MACH_TYPE} \
  --release-channel=${REL_CHANNEL} \
  --enable-vertical-pod-autoscaling # just in case we need it!
```
