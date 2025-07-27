# Configurer un réseau Google Cloud

## Initial config
```bash
gcloud config set compute/zone "<ZONE>"
export ZONE=$(gcloud config get compute/zone)

gcloud config set compute/region "<REGION>"
export REGION=$(gcloud config get compute/region)
```

## Tâche 1 : Créer des réseaux
### Créez le réseau VPC "vpc-network-fl5c"
```bash
gcloud compute networks create vpc-network-fl5c --subnet-mode custom
```

### Créez "subnet-a-pwqh" avec un préfixe IP
```bash
gcloud compute networks subnets create subnet-a-pwqh \
   --network vpc-network-fl5c \
   --region us-east4 \
   --range 10.10.10.0/24
```

### Créez "subnet-b-2kr5" avec un préfixe IP
```bash
gcloud compute networks subnets create subnet-b-2kr5 \
   --network vpc-network-fl5c \
   --region europe-west4 \
   --range 10.10.20.0/24
```

### Affichez la liste de vos réseaux 
```bash
gcloud compute networks subnets list \
   --network taw-custom-network
```

## Tâche 2 : Ajouter des règles de pare-feu

### omti-firewall-ssh
```bash
gcloud compute firewall-rules create omti-firewall-ssh \
--allow tcp:22 --network vpc-network-fl5c --source-ranges 0.0.0.0/0 \
--priority=1000
```

### gtoo-firewall-rdp
```bash
gcloud compute firewall-rules create gtoo-firewall-rdp \
--allow tcp:3389 --network vpc-network-fl5c --source-ranges 0.0.0.0/24 \
--priority=65535
```

### otgy-firewall-icmp
```bash
gcloud compute firewall-rules create otgy-firewall-icmp \
--allow icmp --network vpc-network-fl5c --source-ranges 10.10.10.0/24,10.10.20.0/24 \
--priority=1000
```

## Tâche 3 : Ajouter des VM à votre réseau

### Créez une instance nommée us-test-01 
```bash
gcloud compute instances create us-test-01 \
--subnet subnet-a-pwqh \
--zone us-east4-b \
--machine-type e2-standard-2
```

### Créez une instance nommée us-test-02 
```bash
gcloud compute instances create us-test-02 \
--subnet subnet-b-2kr5 \
--zone europe-west4-c \
--machine-type e2-standard-2
```

