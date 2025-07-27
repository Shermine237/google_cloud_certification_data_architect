# Créer un réseau personnalisé et appliquer des règles de pare-feu

## Initial config
```bash
gcloud config set compute/zone "Zone"
export ZONE=$(gcloud config get compute/zone)

gcloud config set compute/region "Region"
export REGION=$(gcloud config get compute/region)
```

## Créer un réseau personnalisé à l'aide de Cloud Shell
### Créez le réseau personnalisé
```bash
gcloud compute networks create taw-custom-network --subnet-mode custom
```

### Créez "subnet-<REGION>" avec un préfixe IP
```bash
gcloud compute networks subnets create subnet-<REGION> \
   --network taw-custom-network \
   --region Region \
   --range 10.0.0.0/16
```

### Créez "subnet-<REGION>" avec un préfixe IP
```bash
gcloud compute networks subnets create subnet-<REGION> \
   --network taw-custom-network \
   --region Region \
   --range 10.1.0.0/16
```

### Créez "subnet-<REGION>" avec un préfixe IP
```bash
gcloud compute networks subnets create subnet-<REGION> \
   --network taw-custom-network \
   --region Region \
   --range 10.2.0.0/16
```

### Affichez la liste de vos réseaux 
```bash
gcloud compute networks subnets list \
   --network taw-custom-network
```

## Ajouter des règles de pare-feu

### nw101-allow-http au réseau taw-custom-network
```bash
gcloud compute firewall-rules create nw101-allow-http \
--allow tcp:80 --network taw-custom-network --source-ranges 0.0.0.0/0 \
--target-tags http
```

### Créer des règles de pare-feu supplémentaires
ICMP
```bash
gcloud compute firewall-rules create "nw101-allow-icmp" --allow icmp --network "taw-custom-network" --target-tags rules
```

Communications internes
```bash
gcloud compute firewall-rules create "nw101-allow-internal" --allow tcp:0-65535,udp:0-65535,icmp --network "taw-custom-network" --source-ranges "10.0.0.0/16","10.2.0.0/16","10.1.0.0/16"
```

SSH
```bash
gcloud compute firewall-rules create "nw101-allow-ssh" --allow tcp:22 --network "taw-custom-network" --target-tags "ssh"
```

RDP
```bash
gcloud compute firewall-rules create "nw101-allow-rdp" --allow tcp:3389 --network "taw-custom-network"
```

