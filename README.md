# 🐘 Installation d'un Serveur MooseFS Master + Chunkserver
[![MooseFS](https://img.shields.io/badge/MooseFS-Distributed%20FS-red?style=flat-square&logo=linux)](https://moosefs.com/)

## 🧾 Prérequis

* 🖥️ Système : **Debian 12 et Debian 13**
* 🔐 Accès `sudo` ou root
* 🌐 Configuration réseau fonctionnelle

> [!caution]
> ✅ Cette documentation a été **testée et validée** sur une machine virtuelle Proxmox sous **Debian 13**.  
> ❌ Si vous rencontrez des problèmes, vérifiez votre configuration réseau, DNS et vos disques.

> [!note]
> 📌 Cette installation configure un serveur **hybride** qui agit à la fois comme **Master Server** (métadonnées) et **Chunkserver** (stockage de données).

---

## 🧹 (Optionnel) Étendre la partition root

Si vous utilisez Proxmox et avez besoin d'espace supplémentaire :

```bash
lvremove /dev/pve/data
lvresize -l +100%FREE /dev/pve/root
resize2fs /dev/mapper/pve-root
```

---

## 🐂 Ajouter les dépôts MooseFS

```bash
sudo mkdir -p /etc/apt/keyrings
curl https://repository.moosefs.com/moosefs.key | \
  gpg -o /etc/apt/keyrings/moosefs.gpg --dearmor

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/moosefs.gpg] http://repository.moosefs.com/moosefs-4/apt/debian/trixie trixie main" > /etc/apt/sources.list.d/moosefs.list
```

---

## 🔄 Mise à jour du système

```bash
sudo apt update
sudo apt dist-upgrade -y
sudo apt autoremove -y
```

---

## 📦 Installation des paquets nécessaires

### Dépendances système

```bash
sudo apt install -y build-essential libpcap-dev zlib1g-dev libfuse3-dev pkg-config fuse3
```

### Paquets MooseFS (Master + Chunkserver)

```bash
sudo apt install -y moosefs-master moosefs-chunkserver moosefs-metalogger moosefs-client moosefs-cgi moosefs-cgiserv moosefs-cli
```

---

## 🔧 Configuration du Master Server

### 📁 Préparer les répertoires

```bash
sudo mkdir -p /var/lib/mfs
sudo chown -R mfs:mfs /var/lib/mfs
```

### 📝 Configuration du Master

```bash
cd /etc/mfs
sudo cp mfsmaster.cfg.sample mfsmaster.cfg
sudo cp mfsexports.cfg.sample mfsexports.cfg
```

### 🗄️ Initialiser le fichier de métadonnées

```bash
cd /var/lib/mfs
sudo cp metadata.mfs.empty metadata.mfs
sudo chown mfs:mfs metadata.mfs
sudo rm metadata.mfs.empty
```

### ⚙️ (Optionnel) Personnaliser la configuration

```bash
sudo nano /etc/mfs/mfsmaster.cfg
```

Paramètres importants à vérifier :
- `WORKING_USER = mfs`
- `WORKING_GROUP = mfs`
- `DATA_PATH = /var/lib/mfs`

### 🔐 Configuration des exports (permissions d'accès)

```bash
sudo nano /etc/mfs/mfsexports.cfg
```

Exemple de configuration permettant l'accès au réseau local :

```bash
# Autoriser tout le réseau 192.168.1.0/24 en lecture/écriture
192.168.1.0/24          /       rw,alldirs,maproot=0

# Ou autoriser tous les clients (⚠️ moins sécurisé)
*                       /       rw,alldirs,maproot=0
```

---

## 🗂️ Configuration du Chunkserver

### 📁 Préparer les répertoires de stockage

```bash
sudo mkdir -p /mnt/moosefs_chunks
sudo mkdir -p /mnt/moosefs_data
sudo chown -R mfs:mfs /mnt/moosefs_chunks
```

### 📝 Configuration du Chunkserver

```bash
cd /etc/mfs
sudo cp mfschunkserver.cfg.sample mfschunkserver.cfg
sudo cp mfshdd.cfg.sample mfshdd.cfg
```

### 📌 Définir le disque des chunks

```bash
sudo nano /etc/mfs/mfshdd.cfg
```

**Option 1 : Disque dédié (recommandé)**

```bash
/mnt/moosefs_chunks
```

MooseFS utilisera tout l'espace disponible, moins une marge de sécurité.

**Option 2 : Disque partagé avec le système**

```bash
/mnt/moosefs_chunks =100GiB
```

MooseFS limitera son utilisation à 100 GiB (ajustez selon vos besoins).

> [!tip]
> 💡 Il est recommandé d'utiliser **XFS** comme système de fichiers sous-jacent pour les disques destinés au stockage de chunks.

---

## 📇 Configurer la résolution DNS locale

```bash
sudo nano /etc/hosts
```

Ajouter l'entrée pour le Master :

```bash
# MooseFS Master Server
192.168.1.10    npx-1.lan npx-1 mfsmaster
```

> [!important]
> 🔔 Remplacez `192.168.1.10` par l'adresse IP réelle de votre serveur Master.

---

## 🔁 Montage automatique au démarrage

```bash
sudo nano /etc/fstab
```

Ajouter au début du fichier :

```bash
# MooseFS - Montage automatique
mfsmount    /mnt/moosefs_data    fuse    mfsmaster=mfsmaster,mfsport=9421,_netdev,nonempty    0 0
```

---

## 🚀 Démarrage des services

### Recharger la configuration systemd

```bash
sudo systemctl daemon-reload
```

### 🎯 Démarrer le Master Server

```bash
sudo systemctl enable moosefs-master.service
sudo systemctl start moosefs-master.service
sudo systemctl status moosefs-master.service
```

### 💾 Démarrer le Chunkserver

```bash
sudo systemctl enable moosefs-chunkserver.service
sudo systemctl start moosefs-chunkserver.service
sudo systemctl status moosefs-chunkserver.service
```

### 📋 Démarrer le Metalogger (optionnel mais recommandé)

```bash
cd /etc/mfs
sudo cp mfsmetalogger.cfg.sample mfsmetalogger.cfg

sudo systemctl enable moosefs-metalogger.service
sudo systemctl start moosefs-metalogger.service
sudo systemctl status moosefs-metalogger.service
```

### 🌐 Démarrer l'interface Web (CGI)

```bash
sudo systemctl enable moosefs-cgiserv.service
sudo systemctl start moosefs-cgiserv.service
sudo systemctl status moosefs-cgiserv.service
```

---

## 🗂️ Monter le système de fichiers MooseFS

### Montage manuel

```bash
sudo mkdir -p /mnt/moosefs_data
sudo mount -t moosefs mfsmaster: /mnt/moosefs_data
```

Ou avec `mfsmount` :

```bash
sudo mfsmount -H mfsmaster /mnt/moosefs_data
```

### Vérifier le montage

```bash
df -h | grep moosefs
mount | grep moosefs
```

---

## 🖥️ Accéder à l'interface Web de monitoring

Ouvrez votre navigateur et accédez à :

```
http://mfsmaster:9425
```

Ou avec l'adresse IP :

```
http://192.168.1.10:9425
```

Vous pourrez y voir :
- État du cluster
- Espace disque utilisé/disponible
- Nombre de chunks
- Liste des serveurs connectés
- Statistiques de performance

---

## ✅ Vérifications post-installation

### Vérifier l'état du Master

```bash
sudo mfsmaster -v
sudo systemctl status moosefs-master
```

### Vérifier l'état du Chunkserver

```bash
sudo mfschunkserver -v
sudo systemctl status moosefs-chunkserver
```

### Lister les serveurs connectés

```bash
mfscli -SIN
mfscli -SCS
```

### Vérifier l'espace disponible

```bash
df -h /mnt/moosefs_data
```

---

## 🔧 Commandes utiles

### Informations sur le cluster

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

### Gestion des objectifs de réplication

```bash
# Définir un objectif de réplication pour un fichier/dossier
mfssetgoal 2 /mnt/moosefs_data/mon_dossier

# Vérifier l'objectif
mfsgetgoal /mnt/moosefs_data/mon_dossier
```

---

## 🛡️ Recommandations de production

> [!warning]
> ⚠️ Pour un environnement de production :

1. **Haute disponibilité** : Configurez au moins 2 Chunkservers supplémentaires sur des machines séparées
2. **Sauvegarde des métadonnées** : Installez un Metalogger sur une machine différente du Master
3. **Monitoring** : Surveillez régulièrement l'interface CGI et les logs
4. **Objectif de réplication** : Configurez `goal=2` minimum (2 copies de chaque fichier)
5. **Système de fichiers** : Utilisez XFS pour les partitions de chunks
6. **Réseau** : Utilisez un réseau Gigabit ou supérieur
7. **Sauvegardes** : Sauvegardez régulièrement `/var/lib/mfs/metadata.mfs`

---

## 📚 Ressources supplémentaires

- 🌐 Site officiel : [https://moosefs.com](https://moosefs.com)
- 📖 Documentation : [https://moosefs.com/support](https://moosefs.com/support)
- 🐙 GitHub : [https://github.com/moosefs/moosefs](https://github.com/moosefs/moosefs)
- 💬 Support : support@moosefs.com

---

## 📝 Licence

MooseFS est distribué sous **licence GPL v2**.  
Copyright © 2008-2025 Jakub Kruszona-Zawadzki, Saglabs SA
