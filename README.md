# Installation d'un Serveur MooseFS Master + Chunkserver

## Prérequis

* Système d'exploitation : Debian 13
* Accès administrateur (sudo ou root)
* Configuration réseau fonctionnelle

> [!caution]
> Cette documentation a été testée et validée sur une machine virtuelle Proxmox sous Debian 13.  
> En cas de problème, vérifiez votre configuration réseau, DNS et vos disques.

> **Architecture :**  
> Cette installation configure un serveur hybride qui agit à la fois comme Master Server et Chunkserver.

> **Documentation Chunkservers :**  
> Pour l'ajout de Chunkservers, consultez : https://github.com/Mayse-55/MooseFS-Chunkserver
---

## 1. Extension de la partition root (optionnel)

Si vous utilisez Proxmox et nécessitez de l'espace supplémentaire :

```bash
lvremove /dev/pve/data
lvresize -l +100%FREE /dev/pve/root
resize2fs /dev/mapper/pve-root
```

---

## 2. Configuration des dépôts MooseFS

```bash
sudo mkdir -p /etc/apt/keyrings
curl https://repository.moosefs.com/moosefs.key | \
  gpg -o /etc/apt/keyrings/moosefs.gpg --dearmor

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/moosefs.gpg] http://repository.moosefs.com/moosefs-4/apt/debian/trixie trixie main" > /etc/apt/sources.list.d/moosefs.list
```

---

## 3. Mise à jour du système

```bash
sudo apt update
sudo apt dist-upgrade -y
sudo apt autoremove -y
```

---

## 4. Installation des paquets

### 4.1. Dépendances système

```bash
sudo apt install -y build-essential libpcap-dev zlib1g-dev libfuse3-dev pkg-config fuse3
```

### 4.2. Paquets MooseFS

```bash
sudo apt install -y moosefs-master moosefs-chunkserver moosefs-metalogger moosefs-client moosefs-cgi moosefs-cgiserv moosefs-cli
```

---

## 5. Configuration du Master Server

### 5.1. Préparation des répertoires

```bash
sudo mkdir -p /var/lib/mfs
sudo chown -R mfs:mfs /var/lib/mfs
```

### 5.2. Configuration initiale

```bash
cd /etc/mfs
sudo cp mfsmaster.cfg.sample mfsmaster.cfg
sudo cp mfsexports.cfg.sample mfsexports.cfg
```

### 5.3. Initialisation du fichier de métadonnées

```bash
cd /var/lib/mfs
sudo cp metadata.mfs.empty metadata.mfs
sudo chown mfs:mfs metadata.mfs
sudo rm metadata.mfs.empty
```

### 5.4. Personnalisation de la configuration (optionnel)

```bash
sudo nano /etc/mfs/mfsmaster.cfg
```

Paramètres principaux à vérifier :
- `WORKING_USER = mfs`
- `WORKING_GROUP = mfs`
- `DATA_PATH = /var/lib/mfs`

### 5.5. Configuration des permissions d'accès

```bash
sudo nano /etc/mfs/mfsexports.cfg
```

Exemple de configuration pour un réseau local :

```bash
# Autorisation du réseau 192.168.1.0/24 en lecture/écriture
192.168.1.0/24          /       rw,alldirs,maproot=0

# Alternative : autorisation globale (moins sécurisé)
*                       /       rw,alldirs,maproot=0
```

---

## 6. Configuration du Chunkserver

### 6.1. Préparation des répertoires de stockage

```bash
sudo mkdir -p /mnt/moosefs_chunks
sudo mkdir -p /mnt/moosefs_data
sudo chown -R mfs:mfs /mnt/moosefs_chunks
```

### 6.2. Configuration initiale

```bash
cd /etc/mfs
sudo cp mfschunkserver.cfg.sample mfschunkserver.cfg
sudo cp mfshdd.cfg.sample mfshdd.cfg
```

### 6.3. Définition du stockage des chunks

```bash
sudo nano /etc/mfs/mfshdd.cfg
```

**Option A : Disque dédié (recommandé)**

```bash
/mnt/moosefs_chunks
```

MooseFS utilisera tout l'espace disponible avec une marge de sécurité.

**Option B : Disque partagé avec limitation**

```bash
/mnt/moosefs_chunks =100GiB
```

MooseFS limitera son utilisation à 100 GiB.

---

## 7. Configuration de la résolution DNS

```bash
sudo nano /etc/hosts
```

Ajout de l'entrée pour le Master Server :

```bash
# MooseFS Master Server
192.168.1.10    npx-1.lan npx-1 mfsmaster
```

> **Important :** Remplacez `192.168.1.10` par l'adresse IP réelle de votre serveur.

---

## 8. Configuration du montage automatique

```bash
sudo nano /etc/fstab
```

Ajout de la ligne de montage :

```bash
# MooseFS - Montage automatique
mfsmount    /mnt/moosefs_data    fuse    mfsmaster=mfsmaster,mfsport=9421,_netdev,nonempty    0 0
```

---

## 9. Démarrage des services

### 9.1. Rechargement de la configuration systemd

```bash
sudo systemctl daemon-reload
```

### 9.2. Activation du Master Server

```bash
sudo systemctl enable moosefs-master.service
sudo systemctl start moosefs-master.service
sudo systemctl status moosefs-master.service
```

### 9.3. Activation du Chunkserver

```bash
sudo systemctl enable moosefs-chunkserver.service
sudo systemctl start moosefs-chunkserver.service
sudo systemctl status moosefs-chunkserver.service
```

### 9.4. Activation du Metalogger (optionnel mais recommandé)

```bash
cd /etc/mfs
sudo cp mfsmetalogger.cfg.sample mfsmetalogger.cfg

sudo systemctl enable moosefs-metalogger.service
sudo systemctl start moosefs-metalogger.service
sudo systemctl status moosefs-metalogger.service
```

### 9.5. Activation de l'interface Web

```bash
sudo systemctl enable moosefs-cgiserv.service
sudo systemctl start moosefs-cgiserv.service
sudo systemctl status moosefs-cgiserv.service
```

---

## 10. Montage du système de fichiers MooseFS

### 10.1. Montage manuel

```bash
sudo mkdir -p /mnt/moosefs_data
sudo mount -t moosefs mfsmaster: /mnt/moosefs_data
```

Alternative avec mfsmount :

```bash
sudo mfsmount -H mfsmaster /mnt/moosefs_data
```

### 10.2. Vérification du montage

```bash
df -h | grep moosefs
mount | grep moosefs
```

---

## 11. Accès à l'interface Web de monitoring

L'interface est accessible via :

```
http://mfsmaster:9425
```

Ou directement par adresse IP :

```
http://192.168.1.10:9425
```

Informations disponibles :
- État du cluster
- Espace disque utilisé et disponible
- Nombre de chunks
- Liste des serveurs connectés
- Statistiques de performance

---

## 12. Vérifications post-installation

### 12.1. Vérification du Master Server

```bash
sudo mfsmaster -v
sudo systemctl status moosefs-master
```

### 12.2. Vérification du Chunkserver

```bash
sudo mfschunkserver -v
sudo systemctl status moosefs-chunkserver
```

### 12.3. Liste des serveurs connectés

```bash
mfscli -SIN
mfscli -SCS
```

### 12.4. Vérification de l'espace disponible

```bash
df -h /mnt/moosefs_data
```

---

## 13. Informations sur le cluster

```bash
# Informations générales
mfsgetgoal /mnt/moosefs_data

# Statistiques du Master
mfscli -SMI

# Liste des Chunkservers
mfscli -SCS

# État des disques
mfscli -SHD
```

---

## 14. Recommandations pour un environnement de production

### 14.1. Haute disponibilité

Configurez au moins deux Chunkservers supplémentaires sur des machines physiquement séparées.

### 14.2. Sauvegarde des métadonnées

Installez un Metalogger sur une machine différente du Master Server pour assurer la redondance des métadonnées.

### 14.3. Monitoring

Surveillez régulièrement l'interface CGI et les fichiers de logs système pour détecter les anomalies.

### 14.4. Réplication des données

Configurez un objectif de réplication minimum de 2 (deux copies de chaque fichier) pour garantir la disponibilité des données.

### 14.5. Système de fichiers

Utilisez XFS comme système de fichiers sous-jacent pour les partitions dédiées aux chunks.

### 14.6. Infrastructure réseau

Utilisez un réseau Gigabit Ethernet ou supérieur pour assurer des performances optimales.

### 14.7. Sauvegarde régulière

Effectuez des sauvegardes régulières du fichier `/var/lib/mfs/metadata.mfs` sur un support externe.

---

## 15. Ressources et support

- Site officiel : https://moosefs.com
- Documentation : https://moosefs.com/support
- Dépôt GitHub : https://github.com/moosefs/moosefs
- Support technique : support@moosefs.com

---

## 16. Informations légales

MooseFS est distribué sous licence GPL v2.

Copyright 2008-2025 Jakub Kruszona-Zawadzki, Saglabs SA
