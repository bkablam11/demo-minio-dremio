# Guide Complet : Installation Data Lakehouse Local (Dremio + MinIO) sur Docker Windows

Ce guide explique comment monter un environnement Big Data local. Nous utiliserons le dépôt Git d'Alex Merced.
**Objectif :** Connecter **Dremio** (Moteur de requête) à **MinIO** (Stockage) en utilisant le connecteur **Amazon S3**.

---

## 1. Prérequis et Installation

### A. Installer Docker Desktop pour Windows
Si ce n'est pas déjà fait :
1.  Téléchargez l'installateur : [Lien Officiel Docker Desktop](https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe?utm_source=docker&utm_medium=webreferral&utm_campaign=docs-driven-download-win-amd64&_gl=1*j4738o*_gcl_au*MTAxNzQ5MDc3MC4xNzcwODQwNTc5*_ga*MzcwNzQxNDA3LjE3NzA4NDA1Nzg.*_ga_XJWPQMJYHQ*czE3NzEwOTM5NjckbzYkZzEkdDE3NzEwOTM5NzAkajU3JGwwJGgw)
2.  Installez-le en gardant les options par défaut (WSL 2 recommandé).
3.  **Redémarrez votre ordinateur**.

### B. Récupérer le projet
1.  Créez un dossier pour vos projets.
2.  Ouvrez un terminal (PowerShell ou Invite de commandes).
3.  Clonez le dépôt :
    ```bash
    git clone https://github.com/developer-advocacy-dremio/dremio-demo-env-092024.git
    ```
4.  **Entrez dans le dossier** (Étape critique pour éviter l'erreur *"configuration file not found"*) :
    ```bash
    cd dremio-demo-env-092024
    ```

---

## 2. Préparation des Images et Données (Anti-Erreurs)

### A. Téléchargement manuel des images
Pour éviter l'erreur `TLS handshake timeout` due à une connexion lente ou instable, téléchargez les images principales une par une avant de lancer le tout :

```bash
# Image Dremio (avec Superset intégré)
docker pull alexmerced/dremio-superset

# Image MinIO (Stockage)
docker pull minio/minio

# (Optionnel si le script le demande) Image Spark
docker pull alexmerced/spark35nb:latest
```

### B. Placement des données (Seed Data)
Le projet est configuré pour charger automatiquement des fichiers dans MinIO si vous les placez au bon endroit sur votre Windows.

*   **Pour MinIO :** Mettez vos fichiers (CSV, Parquet, JSON) dans le dossier :
    `./minio-data` (situé dans le dossier que vous avez cloné).
*   **Pour Spark :** Mettez vos notebooks dans :
    `./notebook-seed`.

---

## 3. Lancement des Services

Une fois les images téléchargées, lancez l'orchestration :

```bash
docker-compose up -d
```
*L'option `-d` lance les services en arrière-plan.*

> **Vérification :** Ouvrez Docker Desktop. Vous devriez voir un groupe `dremio-demo-env` avec les conteneurs `minio`, `dremio`, `nessie`, etc. allumés en vert.

---

## 4. Configuration de MinIO (Le "Lake")

1.  Accédez à l'interface : **[http://localhost:9001](http://localhost:9001)**
2.  Identifiants :
    *   User : `admin`
    *   Password : `password`
3.  Vérifiez que vos fichiers placés dans le dossier `./minio-data` apparaissent bien dans le bucket. Sinon, créez un bucket nommé `datalake` et uploadez un fichier manuellement.

![Insérer capture d'écran MinIO ici]

---

## 5. Configuration de Dremio (Le "House") - ÉTAPE CRUCIALE

C'est ici que nous connectons Dremio à MinIO en utilisant le protocole S3.

1.  Accédez à l'interface : **[http://localhost:9047](http://localhost:9047)**
2.  Créez votre compte administrateur.
3.  Cliquez sur le bouton **+ Add Source** (en bas à gauche).
4.  Sélectionnez **Amazon S3**.

### Configuration de la source S3 :

#### Onglet "General"
*   **Name :** `MinioData` (ou le nom de votre choix).
*   **Authentication :** AWS Access Key.
*   **Access Key :** `admin`
*   **Secret Key :** `password`

#### Onglet "Advanced Options" (Propriétés de connexion)
Cochez la case **Enable compatibility mode**.

Ajoutez les 3 propriétés suivantes en cliquant sur **Add Property** :

| Nom (Name) | Valeur (Value) | Explication |
| :--- | :--- | :--- |
| **`fs.s3a.path.style.access`** | `true` | Obligatoire pour le mode S3 path-style. |
| **`fs.s3a.endpoint`** | `minio:9000` | **Attention :** Utilisez `minio:9000` (nom du conteneur) et non localhost. |
| **`dremio.s3.compat`** | `true` | Active la compatibilité spécifique Dremio/S3. |

#### Section "Encryption" (Tout en bas)
*   ❌ **DÉCOCHEZ** la case **Encrypt connection**.
    *   *Pourquoi ?* Sinon vous aurez l'erreur `Unsupported or unrecognized SSL message`.

![Insérer capture d'écran configuration Dremio ici]

5.  Cliquez sur **Save**.

---

## 6. Récapitulatif des Accès (URLs)

Voici les adresses pour accéder à tous vos services locaux une fois lancés :

*   **Dremio (Requêtes SQL) :** [http://localhost:9047](http://localhost:9047)
*   **MinIO (Stockage) :** [http://localhost:9001](http://localhost:9001)
*   **Nessie (Catalogue - API uniquement) :** http://localhost:19120
*   **Spark Notebook :** [http://localhost:8888](http://localhost:8888)
*   **Superset (Visualisation) :** [http://localhost:8088](http://localhost:8088)

---

## 7. Gestion du cycle de vie (Arrêt et Nettoyage)

Quand vous avez fini de travailler :

**Pour éteindre les services (les données sont conservées) :**
```bash
docker-compose down
```

**Pour tout effacer (supprime aussi les données et la configuration) :**
```bash
docker-compose down -v
```

---

## 🛠️ Dépannage des erreurs fréquentes

| Erreur | Cause probable | Solution |
| :--- | :--- | :--- |
| `TLS handshake timeout` | Connexion internet saturée lors du téléchargement des images. | Faire les `docker pull` manuellement un par un (voir Étape 2). |
| `no configuration file provided` | Vous n'êtes pas dans le bon dossier. | Faites `cd le_nom_du_dossier` avant de lancer docker-compose. |
| `Unsupported or unrecognized SSL message` | Dremio essaie de parler HTTPS à MinIO HTTP. | **Décochez** "Encrypt connection" dans la source Dremio. |
| `UnknownHostException` | Dremio ne trouve pas le serveur. | Dans `fs.s3a.endpoint`, mettez `minio:9000` au lieu de localhost. |