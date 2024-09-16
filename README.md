# Déploiement d'Odoo sur AWS EC2 avec Docker

Ce guide détaille les étapes nécessaires pour déployer Odoo sur une instance AWS EC2 en utilisant Docker, avec des volumes EBS pour la persistance des données.

## Prérequis

- Une instance AWS EC2 (recommandé : t2.medium ou supérieur)
- Docker et Docker Compose installés sur l'instance
- Volumes EBS attachés à l'instance pour le stockage persistant

## Étapes de configuration

### 1. Préparation de l'instance EC2

1.1. Connectez-vous à votre instance EC2 via SSH.

1.2. Mettez à jour le système et installez Docker :
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose -y
```

1.3. Ajoutez votre utilisateur au groupe docker :
```bash
sudo usermod -aG docker ${USER}
```
Déconnectez-vous et reconnectez-vous pour que les changements prennent effet.

### 2. Configuration des volumes EBS

2.1. Attachez les volumes EBS à votre instance EC2 via la console AWS.

2.2. Identifiez les nouveaux volumes :
```bash
lsblk
```

2.3. Formatez les volumes (si nécessaire) et montez-les :
```bash
sudo mkfs -t ext4 /dev/xvdf
sudo mkfs -t ext4 /dev/xvdg
sudo mkfs -t ext4 /dev/xvdh
```

```bash
sudo mkdir -p /mnt/odoo-data /mnt/odoo-addons /mnt/postgres-data
```

```bash
sudo mount /dev/xvdf /mnt/odoo-data
sudo mount /dev/xvdg /mnt/odoo-addons
sudo mount /dev/xvdh /mnt/postgres-data
```

2.4. Configurez le montage automatique en ajoutant ces lignes à `/etc/fstab`:
```
/dev/xvdf /mnt/odoo-data ext4 defaults,nofail 0 2
/dev/xvdg /mnt/odoo-addons ext4 defaults,nofail 0 2
/dev/xvdh /mnt/postgres-data ext4 defaults,nofail 0 2
```

### 3. Configuration de Docker Compose

3.1. Créez un fichier `docker-compose.yml` :
```bash
nano docker-compose.yml
```

3.2. Ajoutez le contenu suivant :
```yaml
version: '3.1'
services:
  web:
    image: odoo:14.0
    depends_on:
      - db
    ports:
      - "8069:8069"
    volumes:
      - /mnt/odoo-data:/var/lib/odoo
      - /mnt/odoo-addons:/mnt/extra-addons
    environment:
      - HOST=db
      - USER=odoo
      - PASSWORD=myodoopass
    command: -- --db_host db
    restart: unless-stopped
    user: "101:101"
  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_PASSWORD=myodoopass
      - POSTGRES_USER=odoo
    volumes:
      - /mnt/postgres-data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  odoo-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/odoo-data
  odoo-addons:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/odoo-addons
  postgres-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/postgres-data
```

### 4. Configuration des permissions

4.1. Définissez les bonnes permissions pour les répertoires de données :
```bash
sudo chown -R 101:101 /mnt/odoo-data /mnt/odoo-addons
sudo chmod -R 755 /mnt/odoo-data /mnt/odoo-addons
sudo chown -R 999:999 /mnt/postgres-data
sudo chmod -R 700 /mnt/postgres-data
```

### 5. Lancement des conteneurs

5.1. Démarrez les conteneurs :
```bash
docker-compose up -d
```

5.2. Vérifiez que les conteneurs sont en cours d'exécution :
```bash
docker-compose ps
```

### 6. Configuration du pare-feu EC2

6.1. Dans la console AWS, ajoutez une règle entrante au groupe de sécurité de l'instance pour autoriser le trafic sur le port 8069.

### 7. Accès à Odoo

7.1. Accédez à Odoo via un navigateur web en utilisant l'adresse IP publique de votre instance EC2 : `http://<EC2-Public-IP>:8069`

## Dépannage

- Si vous rencontrez des problèmes de permissions, vérifiez les logs :
  ```bash
  docker-compose logs web
  ```

- Pour les problèmes de base de données, vérifiez les logs PostgreSQL :
  ```bash
  docker-compose logs db
  ```

- Si Odoo ne démarre pas, essayez de recréer les conteneurs :
  ```bash
  docker-compose down
  docker-compose up -d
  ```

## Sauvegarde et restauration

### Sauvegarde

Pour sauvegarder vos données Odoo et PostgreSQL, vous pouvez créer des snapshots de vos volumes EBS via la console AWS.

### Restauration

Pour restaurer à partir d'une sauvegarde :
1. Créez de nouveaux volumes à partir des snapshots EBS.
2. Attachez ces volumes à une nouvelle instance EC2.
3. Suivez les étapes de ce guide pour monter les volumes et configurer Docker.

## Gestion des utilisateurs PostgreSQL

Pour gérer les utilisateurs PostgreSQL, connectez-vous au conteneur de base de données :

```bash
docker exec -it <nom_du_conteneur_postgres> psql -U odoo -d postgres
```

Vous pouvez ensuite utiliser des commandes SQL pour gérer les utilisateurs et les permissions.

## Sécurité

- Changez régulièrement les mots de passe des utilisateurs Odoo et PostgreSQL.
- Mettez à jour régulièrement les images Docker et le système d'exploitation de l'instance EC2.
- Configurez des sauvegardes automatiques de vos volumes EBS.
- Utilisez HTTPS pour sécuriser l'accès à Odoo en production.

---

N'hésitez pas à adapter ce guide en fonction de vos besoins spécifiques et de votre environnement.