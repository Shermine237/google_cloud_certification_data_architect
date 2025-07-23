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

### Apply changed wp-env.yaml file:
```bash
kubectl create -f wp-env.yaml
```
## To create key for service account, and add to Kubernetes environment
```bash
gcloud iam service-accounts keys create key.json \
    --iam-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
kubectl create secret generic cloudsql-instance-credentials \
    --from-file key.json
```

## Retrieve connection name
```bash
gcloud sql instances describe griffin-dev-db --format='value(connectionName)'
```

## Update wp-deployment with your sql instance connection name
```bash
sed -i s/YOUR_SQL_INSTANCE/$(gcloud sql instances describe griffin-dev-db --format="value(connectionName)")/g wp-deployment.yaml
```
## Deploy WordPress to Kubernetes
```bash
kubectl create -f wp-deployment.yaml
kubectl create -f wp-service.yaml
```

### Verify and copy the EXTERNAL-IP for wordpress LoadBalancer
```bash
kubectl get deployments
kubectl get services
```
## Enable monitoring : Uptime check
An uptime check verifies that the WordPress development site is up and available.

    - In Google Cloud console, navigate to Navigation menu > Monitoring
    - In the left pane of Cloud Monitoring, click on Uptime checks, and then click Create Uptime Check.
    For Protocol, leave as HTTP.
    For Resource Type, leave as URL.
    For Hostname, paste the External-IP of the wordpress LoadBalancer.
    For Path, enter /.
    - Click Continue.
    - In Response Validation, accept the defaults and then click Continue.
    - In Alert & Notification, accept the defaults, and then click Continue.
    - For Title, type WPress Uptime Check.
    - Click Test to verify that your uptime check can connect to the resource.
    - When you see a green check mark everything can connect.
    - Click Create.

## Provide access for another engineer
```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member="user:$ADDITIONAL_ENGINEER_EMAIL" \
    --role="roles/editor"
```

## Done :)

