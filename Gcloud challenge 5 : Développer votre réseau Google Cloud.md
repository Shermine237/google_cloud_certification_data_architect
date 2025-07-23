# Develop your Google Cloud Network: Challenge Lab Solution

## Variables
```bash
export REGION=<your-region-here>
export ZONE=<your-zone-here>
export ADDITIONAL_ENGINEER_EMAIL=<your additional-student-here>
```

## Create Dev VPC
```bash
gcloud compute networks create griffin-dev-vpc --subnet-mode=custom
```
```bash
gcloud compute networks subnets create griffin-dev-wp \
--network=griffin-dev-vpc \
--range=192.168.16.0/20 \
--region=$REGION
```
```bash
gcloud compute networks subnets create griffin-dev-mgmt \
--network=griffin-dev-vpc \
--range=192.168.32.0/20 \
--region=$REGION
```

## Create Prod VPC
```bash
gcloud compute networks create griffin-prod-vpc --subnet-mode=custom
```
```bash
gcloud compute networks subnets create griffin-prod-wp \
    --network=griffin-prod-vpc \
    --range=192.168.48.0/20 \
    --region=$REGION
```
```bash
gcloud compute networks subnets create griffin-prod-mgmt \
    --network=griffin-prod-vpc \
    --range=192.168.64.0/20 \
    --region=$REGION
```

## Create bastion host
```bash
gcloud compute instances create griffin-bastion \
    --machine-type=e2-medium \
    --zone=$ZONE \
    --tags=bastion \
    --network-interface=subnet=griffin-dev-mgmt \
    --network-interface=subnet=griffin-prod-mgmt \
    --metadata=startup-script='#! /bin/bash
        sudo apt-get update
        sudo apt-get install -yq git htop' \
    --scopes=cloud-platform \
    --image-family=debian-11 \
    --image-project=debian-cloud
```

### Firewalls
```bash
gcloud compute firewall-rules create griffin-dev-allow-ssh \
    --network=griffin-dev-vpc \
    --allow=tcp:22 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=bastion \
    --description="Allow SSH access to bastion host"
```
```bash
gcloud compute firewall-rules create griffin-prod-allow-ssh \
    --network=griffin-prod-vpc \
    --allow=tcp:22 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=bastion \
    --description="Allow SSH access to bastion host in production"
```

## Create Cloud SQL instances
```bash
gcloud sql instances create griffin-dev-db \
    --database-version=MYSQL_5_7 \
    --tier=db-n1-standard-1 \
    --region=$REGION
```
```bash
gcloud sql connect griffin-dev-db --user=root << EOF
CREATE DATABASE wordpress;
CREATE USER 'wp_user'@'%' IDENTIFIED BY 'stormwind_rules';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'%';
FLUSH PRIVILEGES;
EOF
```

## Create Kubernetes Cluster
```bash
gcloud container clusters create griffin-dev \
    --zone=$ZONE \
    --num-nodes=2 \
    --machine-type=e2-standard-4 \
    --network=griffin-dev-vpc \
    --subnetwork=griffin-dev-wp
```

## Copy configuration files
```bash
gsutil cp -r gs://cloud-training/gsp321/wp-k8s . && cd wp-k8s
```

## Update wp-env to use proper user and password
### To configure username to wp_user and password to stormwind_rules, you can use sed command :
```bash
sed -i s/username_goes_here/wp_user/g wp-env.yaml
sed -i s/password_goes_here/stormwind_rules/g wp-env.yaml
```

gcloud iam service-accounts keys create key.json \
    --iam-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com

kubectl create secret generic cloudsql-instance-credentials \
    --from-file key.json

# Retrieve connection name

gcloud sql instances describe griffin-dev-db --format='value(connectionName)'

# Update wp-deployment with your sql instance connection name
# Review changes of those files usign ‘cat’
# Deploy WordPress to Kubernetes

kubectl apply -f wp-env.yaml
kubectl apply -f wp-deployment.yaml
kubectl apply -f wp-service.yaml

# Set variable for WP Site URL

export WORDPRESS_SITE_URL="IPgoesHere"

# Uptime check

gcloud monitoring uptime create griffin-dev-wp-uptime-check \
    --display-name="Griffin Dev WP Uptime Check" \
    --resource-labels=host=$WORDPRESS_EXTERNAL_IP

# Provide access for another engineer

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member="user:$ADDITIONAL_ENGINEER_EMAIL" \
    --role="roles/editor"
