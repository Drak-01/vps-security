# Préparation du serveur

## Updating system

```bash
sudo apt update

sudo apt upgrade
```

## Create and use an SSH key 

```bash
ssh-keygen -t rsa -b 4096 -a 100
ssh-keygen -t rsa -b 4096 -C "votre_email@exemple.com"

cat ~/.ssh/id_rsa.pub

ssh-copy-id -i ~/.ssh/your-key.pub username@ip

# Test connexion
ssh -i ~/.ssh/your-vps username@IP
```

