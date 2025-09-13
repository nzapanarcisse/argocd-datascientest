# Cours Complet ArgoCD : De la Théorie à la Production avec GitOps

![ArgoCD Logo](https://raw.githubusercontent.com/argoproj/argo-cd/master/docs/assets/argo.png)

Bienvenue dans ce cours complet sur ArgoCD. En tant que professionnels du DevOps, nous savons que l'automatisation, la fiabilité et la cohérence sont les piliers de nos déploiements. Ce cours est conçu comme un atelier pratique pour vous faire découvrir la puissance du GitOps avec ArgoCD, un outil qui révolutionne la livraison continue sur Kubernetes.

## Table des Matières

1.  [**Introduction : Pourquoi ArgoCD dans une démarche DevOps ?**](#1--introduction--pourquoi-argocd-dans-une-démarche-devops-)
    *   Les limites du CI/CD traditionnel (Push)
    *   Le GitOps comme "Single Source of Truth" (Pull)
    *   Les bénéfices d'ArgoCD
2.  [**Architecture et Fonctionnement**](#2--architecture-et-fonctionnement)
    *   Les 3 composants clés d'ArgoCD
    *   Le cycle de réconciliation
3.  [**Atelier 1 : Installation de l'Environnement de Travail**](#3--atelier-1--installation-de-lenvironnement-de-travail)
    *   Prérequis
    *   Installation de K3s sur Amazon Linux
    *   Installation d'ArgoCD avec Helm
    *   Accès à l'interface ArgoCD
4.  [**Atelier 2 : Déploiement d'Applications avec ArgoCD**](#4--atelier-2--déploiement-dapplications-avec-argocd)
    *   **Cas 1 : Déploiement d'Odoo 18 via l'UI (Chart Helm Existant)**
    *   **Cas 2 : Préparation de notre application "Site Vitrine"**
        *   Création du code source et des manifestes Kubernetes
        *   Création du dépôt GitHub
    *   **Cas 3 : Déploiement du "Site Vitrine" via l'UI (Dépôt Git Privé)**
5.  [**Atelier 3 : Workflow CI/CD Complet avec GitHub Actions**](#5--atelier-3--workflow-cicd-complet-avec-github-actions)
    *   Le concept : Git est la seule source de vérité
    *   Mise en place du pipeline GitHub Actions
    *   Démonstration du workflow automatisé
6.  [**Concepts Avancés**](#6--concepts-avancés)
    *   **ArgoCD et le Multi-Cluster**
        *   Enregistrer un second cluster
        *   Déployer sur un cluster distant
    *   **Notifications avec Slack**
        *   Configuration du webhook Slack
        *   Mise à jour de notre application pour les notifications
7.  [**Conclusion**](#7--conclusion)
8.  [**Examen de Validation : Déployez votre Propre Application**](#8--examen-de-validation--déployez-votre-propre-application)


---

## 1. Introduction : Pourquoi ArgoCD dans une démarche DevOps ?

Dans un pipeline CI/CD "classique", c'est souvent le serveur de CI (Jenkins, GitLab CI, GitHub Actions) qui, après avoir construit une image, la "pousse" vers Kubernetes à l'aide de `kubectl apply` ou `helm upgrade`.

**Cette approche (Push) présente des défis :**
*   **Sécurité :** Le système de CI a besoin de credentials puissants pour accéder au cluster Kubernetes. C'est une surface d'attaque importante.
*   **Cohérence :** Qu'est-ce qui est réellement déployé sur le cluster ? L'état du cluster peut dériver de ce qui est dans Git si des modifications manuelles (`kubectl edit`) sont faites.
*   **Audit :** Il est difficile de savoir qui a changé quoi et quand directement depuis l'état du cluster.

**Le GitOps propose un modèle inverse (Pull) :**
Le dépôt **Git est la seule et unique source de vérité** (`Single Source of Truth`). L'état désiré de notre infrastructure est décrit de manière déclarative dans Git (fichiers YAML Kubernetes, charts Helm, etc.).

**ArgoCD** est un agent qui tourne dans notre cluster et dont le rôle est de s'assurer que l'état *réel* du cluster correspond en permanence à l'état *désiré* dans Git.

**Bénéfices immédiats :**
*   **Déploiements fiables et reproductibles :** Tout est dans Git. Pour revenir en arrière, il suffit de faire un `git revert`.
*   **Sécurité renforcée :** Le cluster n'expose plus ses credentials. C'est ArgoCD, à l'intérieur, qui tire les changements.
*   **Détection de la dérive (Drift Detection) :** Toute modification manuelle sur le cluster est immédiatement détectée (et peut être automatiquement annulée).
*   **Expérience développeur améliorée :** Les développeurs n'ont plus besoin de connaître `kubectl`. Ils font ce qu'ils savent faire : une `pull request`.

## 2. Architecture et Fonctionnement

ArgoCD est simple mais puissant. Il repose sur 3 composants principaux :

1.  **API Server :** C'est la porte d'entrée. Elle expose une API gRPC/REST qui est utilisée par l'interface web (UI), la CLI (`argocd`), et les webhooks. Elle gère l'authentification, les autorisations et la coordination des autres services.

2.  **Repository Server :** Ce service est responsable de cloner vos dépôts Git en local et de générer les manifestes Kubernetes. Il met en cache les dépôts pour éviter de surcharger Git. C'est lui qui transforme un Chart Helm ou un fichier Kustomize en YAML Kubernetes pur.

3.  **Application Controller :** C'est le cœur d'ArgoCD. Ce contrôleur Kubernetes surveille en permanence les applications déployées. Pour chaque application, il compare l'état désiré (les manifestes générés par le Repository Server depuis Git) avec l'état actuel dans le cluster. Si une différence est détectée, l'application est marquée comme `OutOfSync`. Le contrôleur peut alors (selon la configuration) corriger l'état du cluster pour qu'il corresponde à Git.

Ce cycle de comparaison et de correction est appelé le **cycle de réconciliation**.

## 3. Atelier 1 : Installation de l'Environnement de Travail

### Prérequis
*   Un compte AWS avec un serveur Amazon Linux 2 ou 2023.
*   Un client SSH pour vous connecter à votre instance.
*   [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) installé sur votre machine locale.
*   [Helm](https://helm.sh/docs/intro/install/) installé sur votre machine locale.

### Installation de K3s sur Amazon Linux

K3s est une distribution Kubernetes légère et certifiée, parfaite pour nos ateliers. Connectez-vous en SSH à votre instance EC2 et exécutez :

```bash
# Installation de K3s
curl -sfL https://get.k3s.io | sh -

# K3s installe son propre containerd, mais il peut y avoir des conflits si Docker est déjà là.
# Cette commande assure que le service k3s est bien démarré.
sudo systemctl start k3s

# Attendre quelques instants que le serveur soit prêt.
# Vérifier que le cluster est fonctionnel (depuis le serveur)
sudo k3s kubectl get nodes
```

Pour piloter le cluster depuis votre machine locale, récupérez le fichier de configuration :

```bash
# Sur le serveur EC2
sudo cat /etc/rancher/k3s/k3s.yaml
```

Copiez le contenu de ce fichier. Sur votre machine locale, collez-le dans `~/.kube/config`. **Attention :** Modifiez l'IP `127.0.0.1` par l'adresse IP publique de votre instance EC2.

```yaml
# Exemple de fichier ~/.kube/config modifié
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <DONNEES_CERTIFICAT>
    server: https://<IP_PUBLIQUE_EC2>:6443 # <-- MODIFIEZ ICI
  name: default
...
```

Testez la connexion depuis votre machine locale :
```bash
kubectl get nodes
# Vous devriez voir le nœud de votre serveur EC2.
```
### Installation helm
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl -y
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm version
# Vous devriez voir la version de votre helm installé.
```
<img width="1215" height="50" alt="image" src="https://github.com/user-attachments/assets/81ce7b42-b4f1-4a84-a2f9-7c296a18f3c9" />

### Installation d'ArgoCD avec Helm

Nous allons installer ArgoCD dans son propre namespace.

```bash
# 1. Créer le namespace pour ArgoCD
kubectl create namespace argocd

# 2. Ajouter le dépôt Helm d'ArgoCD
helm repo add argo https://argoproj.github.io/argo-helm

# 3. Mettre à jour les dépôts
helm repo update

# 4. Installer ArgoCD
helm install argocd argo/argo-cd --namespace argocd --kubeconfig /etc/rancher/k3s/k3s.yaml
# vérification installation Argocd
kubectl get pod -n argocd
```
<img width="1283" height="364" alt="image" src="https://github.com/user-attachments/assets/e424f1ad-1490-4129-9b93-6afc44ff8eeb" />
<img width="862" height="175" alt="image" src="https://github.com/user-attachments/assets/43643681-52a9-48e8-a50d-0bad80733f26" />

### Accès à l'interface ArgoCD

Pour des raisons de sécurité, le serveur d'API d'ArgoCD n'est pas exposé par défaut. Nous allons utiliser le port-forwarding pour y accéder.

```bash
# Dans un nouveau terminal, lancez cette commande. Elle doit rester active.
kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:443
```

Ouvrez votre navigateur et allez sur `[https://VOTRE_IP_PUBLIC:8080](https://52.23.201.142:8080)`. Ignorez l'avertissement de sécurité (le certificat est auto-signé).
<img width="1286" height="681" alt="image" src="https://github.com/user-attachments/assets/b753a849-4a74-4a38-94f4-d9daa372a3ad" />

Le nom d'utilisateur est `admin`. Pour obtenir le mot de passe initial :

```bash
# Ce mot de passe est stocké dans un secret Kubernetes
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
<img width="1226" height="48" alt="image" src="https://github.com/user-attachments/assets/589a462b-5d3b-4c80-be01-1d70f4deea01" />

Connectez-vous. Vous êtes sur le tableau de bord d'ArgoCD !
<img width="1273" height="692" alt="image" src="https://github.com/user-attachments/assets/d3d32f92-20bc-4e88-81b0-6211a8486f60" />

## 4. Atelier 2 : Déploiement d'Applications avec ArgoCD

### Cas 1 : Déploiement d'Odoo 18 via l'UI (Chart Helm Existant)

C'est le cas le plus simple : nous voulons déployer une application disponible sur un dépôt de charts Helm public.

1.  **Cliquez sur `+ NEW APP`** sur l'interface d'ArgoCD.
2.  Remplissez le formulaire :
    *   **Application Name** : `odoo`
    *   **Project Name** : `default`
    *   **Sync Policy** : `Automatic`
        *   Cochez `Prune Resources` (supprime les ressources qui ne sont plus dans Git).
        *   Cochez `Self Heal` (annule les changements manuels sur le cluster).
3.  **Source** :
    *   **Repository URL** : `https://charts.bitnami.com/bitnami`
    *   **Chart** : `odoo`
    *   **Version** : `20.2.2` (ou une version récente)
4.  **Destination** :
    *   **Cluster URL** : `https://kubernetes.default.svc` (le cluster local)
    *   **Namespace** : `odoo`
        *   Nous allons créer ce namespace. Dans la section **Helm**, cliquez sur `Parameters` et ajoutez un paramètre :
        *   **Name**: `namespace.create`, **Value**: `true`
5.  **Cliquez sur `CREATE`** en haut.

ArgoCD va créer l'application, la marquer comme `OutOfSync` (car elle n'existe pas encore sur le cluster), puis la synchroniser automatiquement. En quelques minutes, vous verrez l'application passer à l'état `Healthy` et `Synced`.

Vous pouvez voir les ressources Kubernetes créées (Deployments, Services, PVC...) directement dans l'interface.

### Cas 2 : Préparation de notre application "Site Vitrine"

Maintenant, nous allons déployer notre propre application. Le code se trouve dans le dossier `app/`.

#### Création du code source et des manifestes Kubernetes

Créez la structure de fichiers suivante sur votre machine :

```
argocd-course/
├── app/
│   ├── index.html
│   └── style.css
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── Dockerfile
```

**`app/index.html`**:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>ArgoCD Rocks!</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <h1>Bienvenue sur notre Site Vitrine !</h1>
        <p>Cette application est déployée avec la magie de GitOps et ArgoCD.</p>
        <p class="version">Version: 1.0.0</p>
    </div>
</body>
</html>
```

**`app/style.css`**:
```css
body { font-family: sans-serif; background: #f0f8ff; text-align: center; }
.container { max-width: 800px; margin: 50px auto; padding: 20px; background: white; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); }
h1 { color: #333; }
.version { font-size: 0.8em; color: #888; }
```

**`Dockerfile`**:
```dockerfile
# Stage 1: Utiliser une image Nginx pour servir nos fichiers statiques
FROM nginx:1.25-alpine

# Copier les fichiers de notre site web dans le répertoire servi par Nginx
COPY app/ /usr/share/nginx/html

# Exposer le port 80
EXPOSE 80

# La commande par défaut de l'image Nginx est déjà de démarrer le serveur
```

**`k8s/deployment.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: site-vitrine-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: site-vitrine
  template:
    metadata:
      labels:
        app: site-vitrine
    spec:
      containers:
      - name: site-vitrine
        # REMPLACEZ par votre nom d'utilisateur DockerHub/GHCR
        image: VOTRE_USER/site-vitrine:latest
        ports:
        - containerPort: 80
```

**`k8s/service.yaml`**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: site-vitrine-service
spec:
  selector:
    app: site-vitrine
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort # Simple pour le test, en production on utiliserait un Ingress
```

#### Création du dépôt GitHub

1.  Créez un nouveau dépôt **privé** sur GitHub (ex: `argocd-site-vitrine`).
2.  Poussez le contenu de votre dossier `argocd-course` dans ce nouveau dépôt.

```bash
# Dans le dossier argocd-course
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/VOTRE_USER/argocd-site-vitrine.git
git push -u origin main
```

### Cas 3 : Déploiement du "Site Vitrine" via l'UI (Dépôt Git Privé)

1.  **Connecter votre dépôt Git à ArgoCD :**
    *   Allez dans `Settings` > `Repositories`.
    *   Cliquez sur `CONNECT REPO USING HTTPS`.
    *   **Type**: `git`
    *   **Project**: `default`
    *   **Repository URL**: L'URL HTTPS de votre dépôt (`https://github.com/VOTRE_USER/argocd-site-vitrine.git`)
    *   **Username**: Votre nom d'utilisateur GitHub.
    *   **Password**: Un **Personal Access Token (PAT)** GitHub. Créez-en un [ici](https://github.com/settings/tokens) avec les droits `repo`.
    *   Cliquez sur `CONNECT`. ArgoCD va tester la connexion.

2.  **Créer l'application :**
    *   Cliquez sur `+ NEW APP`.
    *   **Application Name** : `site-vitrine`
    *   **Project Name** : `default`
    *   **Sync Policy** : `Automatic`, `Prune Resources`, `Self Heal`.
    *   **Source** :
        *   Sélectionnez votre dépôt dans la liste déroulante.
        *   **Revision** : `HEAD`
        *   **Path** : `k8s` (le dossier où se trouvent nos manifestes)
    *   **Destination** :
        *   **Cluster URL** : `https://kubernetes.default.svc`
        *   **Namespace** : `site-vitrine`
    *   Cliquez sur `CREATE`.

**Problème :** Le déploiement va échouer (`ImagePullBackOff`). Pourquoi ? Parce que nous n'avons pas encore construit et poussé l'image Docker `VOTRE_USER/site-vitrine:latest`. C'est l'objet de notre prochain atelier !

## 5. Atelier 3 : Workflow CI/CD Complet avec GitHub Actions

Notre but : quand nous poussons un changement sur la branche `main`, une GitHub Action doit automatiquement :
1.  Construire l'image Docker.
2.  La pousser sur un registre (Docker Hub ou GHCR).
3.  **Mettre à jour le fichier `deployment.yaml` dans Git** avec le nouveau tag de l'image.
4.  ArgoCD détectera ce changement et déploiera la nouvelle version.

### Mise en place du pipeline GitHub Actions

1.  **Ajoutez des secrets à votre dépôt GitHub :**
    *   Allez dans `Settings` > `Secrets and variables` > `Actions`.
    *   Créez les secrets suivants :
        *   `DOCKERHUB_USERNAME`: Votre nom d'utilisateur Docker Hub.
        *   `DOCKERHUB_TOKEN`: Un token d'accès de Docker Hub.

2.  **Créez le fichier de workflow :**
    *   Dans votre dépôt, créez le dossier `.github/workflows/`.
    *   À l'intérieur, créez un fichier `ci-cd.yml`.

**`.github/workflows/ci-cd.yml`**:
```yaml
name: CI/CD for Site Vitrine

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name="Set image tag"
      id: image_tag
      run: echo "tag=$(echo $GITHUB_SHA | head -c7)" >> $GITHUB_OUTPUT

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/site-vitrine:${{ steps.image_tag.outputs.tag }}, ${{ secrets.DOCKERHUB_USERNAME }}/site-vitrine:latest

    - name: Update Kubernetes manifest
      run: |
        # Remplacer le tag de l'image dans le fichier de déploiement
        sed -i 's|image: .*|image: ${{ secrets.DOCKERHUB_USERNAME }}/site-vitrine:${{ steps.image_tag.outputs.tag }}|' k8s/deployment.yaml

    - name: Commit and push changes
      run: |
        git config --global user.name 'GitHub Actions'
        git config --global user.email 'actions@github.com'
        git add k8s/deployment.yaml
        # Vérifier s'il y a des changements à commiter
        git diff --staged --quiet || git commit -m "Update image tag to ${{ steps.image_tag.outputs.tag }}"
        git push
```

### Démonstration du workflow automatisé

1.  **Poussez le fichier de workflow** sur votre branche `main`.
2.  Allez dans l'onglet `Actions` de votre dépôt GitHub. Vous verrez le workflow s'exécuter. Il va construire, pousser l'image, puis commiter un changement dans `k8s/deployment.yaml`.
3.  Retournez sur l'interface d'ArgoCD. Cliquez sur le bouton `Refresh` de l'application `site-vitrine`. ArgoCD va détecter que le commit dans Git a changé.
4.  L'application passera à `OutOfSync`. Comme la politique de synchronisation est `Automatic`, ArgoCD va automatiquement appliquer le nouveau manifeste, créant un nouveau ReplicaSet avec les pods utilisant la nouvelle image.
5.  **Faites un changement !** Modifiez le fichier `app/index.html` (par exemple, changez la version en "2.0.0").
6.  `git commit` et `git push`.
7.  Observez la magie : GitHub Actions se déclenche, met à jour l'image et le manifeste. ArgoCD voit le changement et déploie la nouvelle version. **Vous venez de mettre en place un vrai workflow GitOps !**

## 6. Concepts Avancés

### ArgoCD et le Multi-Cluster

ArgoCD peut gérer des déploiements sur plusieurs clusters Kubernetes.

#### Enregistrer un second cluster

Supposons que vous ayez un second cluster (par exemple, un cluster de `staging`).
1.  Assurez-vous que votre `~/.kube/config` local contient les contextes pour les deux clusters.
2.  Utilisez la CLI d'ArgoCD (que vous pouvez installer localement) pour ajouter le nouveau cluster.

```bash
# Lister les contextes de votre kubeconfig
kubectl config get-contexts

# Ajouter un nouveau cluster à ArgoCD
# Remplacez <CONTEXT_NAME> par le nom du contexte de votre cluster de staging
argocd cluster add <CONTEXT_NAME>
```
ArgoCD va créer un ServiceAccount et un ClusterRoleBinding sur le cluster distant pour pouvoir y gérer les ressources.

#### Déployer sur un cluster distant

Pour déployer notre `site-vitrine` sur ce nouveau cluster, il suffit de changer la destination :
*   Soit en modifiant l'application dans l'UI et en choisissant le nouveau cluster dans la liste déroulante `Destination`.
*   Soit, de manière plus GitOps, en définissant l'application via un manifeste YAML.

**`app-site-vitrine-staging.yaml`**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: site-vitrine-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/VOTRE_USER/argocd-site-vitrine.git'
    path: k8s
    targetRevision: HEAD
  destination:
    # Nom du cluster tel qu'affiché par `argocd cluster list`
    name: <NOM_DU_CLUSTER_STAGING> 
    namespace: site-vitrine
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
Appliquez ce fichier à votre cluster principal (celui où tourne ArgoCD) : `kubectl apply -f app-site-vitrine-staging.yaml`. ArgoCD déploiera alors l'application sur le cluster de staging.

### Notifications avec Slack

Recevoir des alertes sur l'état de nos déploiements est crucial.

1.  **Créer un Webhook entrant sur Slack :**
    *   Allez sur `api.slack.com/apps`.
    *   Créez une nouvelle application pour votre workspace.
    *   Activez `Incoming Webhooks` et créez un nouveau webhook pour le canal de votre choix (ex: `#deployments`).
    *   Copiez l'URL du webhook.

2.  **Configurer ArgoCD Notifications :**
    *   Nous devons stocker l'URL du webhook dans un secret.

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Secret
    metadata:
      name: argocd-notifications-secret
      namespace: argocd
    stringData:
      slack-url: <VOTRE_URL_DE_WEBHOOK_SLACK>
    EOF
    ```

    *   Modifiez le ConfigMap `argocd-notifications-cm` pour définir le service Slack.

    ```bash
    kubectl patch configmap argocd-notifications-cm -n argocd --type merge -p '{"data": {"service.slack": "{url: $slack-url}"}}'
    ```

3.  **S'abonner aux notifications :**
    *   Il suffit d'ajouter une annotation à notre application ArgoCD.

    ```bash
    kubectl patch app site-vitrine -n argocd -p '{"metadata": {"annotations": {"notifications.argoproj.io/subscribe.on-sync-succeeded.slack": "VOTRE_CANAL_SLACK"}}}' --type merge
    ```
    *   Cette annotation dit : "Quand la synchronisation réussit (`on-sync-succeeded`), envoie une notification via le service `slack` au canal `VOTRE_CANAL_SLACK`".

Maintenant, à chaque déploiement réussi, vous recevrez une belle notification sur Slack !

## 7. Conclusion

Félicitations ! Vous avez parcouru le cycle de vie complet d'une application gérée avec ArgoCD. Vous avez :
*   Compris les **principes fondamentaux du GitOps**.
*   Installé un **environnement Kubernetes et ArgoCD fonctionnel**.
*   Déployé des applications depuis des **dépôts publics et privés**.
*   Construit un **pipeline CI/CD complet et automatisé** avec GitHub Actions.
*   Abordé des concepts avancés comme le **multi-cluster et les notifications**.

ArgoCD est un outil extrêmement riche. N'hésitez pas à explorer d'autres fonctionnalités comme les **Sync Waves** (pour ordonner les déploiements), les **Hooks** (pour des actions pré/post-synchronisation), ou son intégration avec **Kustomize**.

Le GitOps n'est pas juste un outil, c'est une méthodologie qui apporte rigueur, sécurité et vélocité à vos déploiements. ArgoCD en est l'une des meilleures implémentations. Bon déploiement !

## 8. Examen de Validation : Déployez votre Propre Application

Cet examen a pour but de valider votre capacité à appliquer les principes GitOps avec ArgoCD de manière autonome. Vous devrez déployer une nouvelle application, diagnostiquer un problème et le corriger en suivant la méthodologie du cours.

### Objectif

Déployer une application web "Citation du Jour" basée sur Python/Flask. L'application est intentionnellement mal configurée. Vous devrez utiliser les outils d'ArgoCD pour trouver le problème, le corriger dans Git, et laisser ArgoCD synchroniser la solution.

### Contexte

L'application est un simple serveur web qui affiche une citation aléatoire à chaque fois que sa page d'accueil est rafraîchie.

**Code source de l'application (`app.py`) :**
```python
from flask import Flask
import random

app = Flask(__name__)

quotes = [
    "The only way to do great work is to love what you do. - Steve Jobs",
    "Innovation distinguishes between a leader and a follower. - Steve Jobs",
    "Your time is limited, don't waste it living someone else's life. - Steve Jobs",
    "Stay hungry, stay foolish. - Steve Jobs",
    "Talk is cheap. Show me the code. - Linus Torvalds"
]

@app.route('/')
def get_quote():
    return f"<h1>Quote of the Day</h1><p>{random.choice(quotes)}</p>"

if __name__ == '__main__':
    # L'application tourne sur le port 5000
    app.run(host='0.0.0.0', port=5000)
```

### Étapes à Réaliser

1.  **Créer la Structure du Projet :**
    *   Créez un nouveau dossier pour votre projet d'examen.
    *   À l'intérieur, créez un fichier `app.py` avec le code ci-dessus.
    *   Créez un fichier `requirements.txt` contenant une seule ligne : `Flask`.

2.  **Écrire le `Dockerfile` :**
    *   Créez un `Dockerfile` pour conteneuriser cette application Python.
    *   *Indice :* Utilisez une image de base Python (ex: `python:3.9-slim`), copiez les fichiers, installez les dépendances via `pip install -r requirements.txt`, et définissez la commande de démarrage.

3.  **Écrire les Manifestes Kubernetes :**
    *   Créez un dossier `k8s/`.
    *   À l'intérieur, créez un `service.yaml` (de type `NodePort` pour simplifier).
    *   Créez un `deployment.yaml`. **Utilisez le manifeste ci-dessous qui contient une erreur volontaire.**

    **`k8s/deployment.yaml` (avec erreur) :**
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: quote-app-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: quote-app
      template:
        metadata:
          labels:
            app: quote-app
        spec:
          containers:
          - name: quote-app
            # Assurez-vous de remplacer par votre image
            image: VOTRE_USER/quote-app:latest
            ports:
            # Attention, il y a une erreur dans cette section
            - containerPort: 8080
    ```

4.  **Mettre en place le Dépôt Git :**
    *   Créez un nouveau dépôt **privé** sur GitHub pour cet examen.
    *   Poussez tous vos fichiers (`app.py`, `requirements.txt`, `Dockerfile`, `k8s/`) sur ce dépôt.

5.  **Construire et Pousser l'Image Docker :**
    *   Construisez l'image Docker de votre application.
    *   Poussez-la sur un registre public (Docker Hub, GHCR...).
    *   Assurez-vous que le nom de l'image dans `deployment.yaml` correspond à celle que vous avez poussée. Poussez ce changement si nécessaire.

6.  **Déployer avec ArgoCD :**
    *   Connectez votre nouveau dépôt à ArgoCD (si ce n'est pas déjà fait via un PAT).
    *   Créez une nouvelle `Application` dans ArgoCD pointant vers le dossier `k8s` de votre dépôt d'examen.
    *   Utilisez une politique de synchronisation `Automatic` avec `Self Heal` et `Prune Resources`.

7.  **Diagnostiquer et Corriger :**
    *   Une fois l'application synchronisée, vous remarquerez qu'elle est `Progressing` mais pas `Healthy`. Les pods vont probablement entrer en `CrashLoopBackOff`.
    *   **Votre mission :** Utilisez l'interface d'ArgoCD pour inspecter l'état de l'application.
        *   Cliquez sur le pod pour voir ses logs et ses événements. Les logs devraient indiquer que le serveur démarre bien sur le port 5000. Les événements du pod ou du ReplicaSet peuvent indiquer un échec des sondes de santé (health checks).
        *   Identifiez l'incohérence entre le port exposé par le conteneur (`5000`) et le port déclaré dans le `deployment.yaml` (`8080`).
    *   **Correction :**
        *   Modifiez le fichier `k8s/deployment.yaml` **dans votre dépôt Git** pour corriger le `containerPort`.
        *   Poussez le commit de correction.
        *   Observez ArgoCD détecter le changement, se re-synchroniser et voir l'application devenir enfin `Healthy`.

### Critères de Réussite

Vous avez réussi l'examen si :
*   ✅ L'application `quote-app` est visible dans ArgoCD.
*   ✅ L'application est `Synced` et `Healthy`.
*   ✅ Le commit de correction de `deployment.yaml` est visible dans l'historique Git de votre dépôt.
*   ✅ Vous pouvez accéder à l'application via son `NodePort` et voir une citation s'afficher dans votre navigateur.

<details>
  <summary>Cliquez ici pour voir la solution</summary>
  
  ---
  
  **Le Problème :**
  
  Le code de l'application Flask (`app.py`) démarre un serveur sur le port `5000`. Cependant, le manifeste `deployment.yaml` déclare que le conteneur expose le port `8080`. Kubernetes ne peut donc pas router correctement le trafic ou vérifier la santé du pod sur le bon port, ce qui cause le crash.
  
  **La Solution :**
  
  Il fallait corriger le `containerPort` dans le fichier `k8s/deployment.yaml` pour qu'il corresponde au port de l'application.
  
  **`k8s/deployment.yaml` (corrigé) :**
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: quote-app-deployment
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: quote-app
    template:
      metadata:
        labels:
          app: quote-app
      spec:
        containers:
        - name: quote-app
          image: VOTRE_USER/quote-app:latest # Votre image
          ports:
          # Le port est maintenant correct
          - containerPort: 5000 
  ```
  En poussant ce changement sur Git, ArgoCD détecte la mise à jour, applique le nouveau manifeste, et le pod démarre correctement, rendant l'application `Healthy`.
  
</details>
