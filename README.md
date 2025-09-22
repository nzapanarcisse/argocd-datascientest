# Cours Complet ArgoCD : De la Théorie à la Production avec GitOps

![ArgoCD Logo](https://raw.githubusercontent.com/argoproj/argo-cd/master/docs/assets/argo.png)

🎯 Présentation du cours : ArgoCD en pratique

Bienvenue dans ce cours pratique sur ArgoCD !

En tant que professionnels du DevOps, nous savons que la réussite de nos déploiements repose sur trois piliers : automatisation, fiabilité et cohérence.
Ce programme a été pensé comme un atelier concret et immersif, pour vous initier à l’approche GitOps et découvrir comment ArgoCD transforme la livraison continue sur Kubernetes.

🚀 Pourquoi suivre ce cours ?

- Comprendre et appliquer le paradigme GitOps au sein de vos projets.

- Déployer et gérer vos applications Kubernetes avec simplicité et efficacité.

- Mettre en place une chaîne CI/CD moderne grâce à ArgoCD et GitHub Actions.

- Gagner en autonomie sur la gestion de vos environnements (dev, staging, production).

🎓 Objectifs pédagogiques

- À l’issue de ce cours, vous serez capable de :

- Installer et configurer ArgoCD dans un cluster Kubernetes.

- Déclarer et synchroniser vos applications à partir de dépôts Git.

- Automatiser vos déploiements en appliquant les bonnes pratiques GitOps.

- Construire un pipeline CI/CD complet avec GitHub Actions et ArgoCD.

👥 Public cible

- Administrateurs systèmes et ingénieurs DevOps.

- Développeurs souhaitant industrialiser leurs déploiements Kubernetes.

- Toute personne désirant maîtriser l’approche GitOps dans un contexte professionnel.

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

# 4. Installer ArgoCD avec les options de ligne de commande
helm install argocd argo/argo-cd --namespace argocd --kubeconfig /etc/rancher/k3s/k3s.yaml \
  --set server.service.type=NodePort

# Vérification de l'installation d'ArgoCD
kubectl get pod -n argocd
```
<img width="1283" height="364" alt="image" src="https://github.com/user-attachments/assets/e424f1ad-1490-4129-9b93-6afc44ff8eeb" />
<img width="862" height="175" alt="image" src="https://github.com/user-attachments/assets/43643681-52a9-48e8-a50d-0bad80733f26" />

### Accès à l'interface ArgoCD

Vérifiez que le service créé par votre stack Argo est bien exposé sur le port 30080.
Ouvrez votre navigateur et allez sur `[https://VOTRE_IP_PUBLIC:30080](https://54.234.45.177:30080)`. Ignorez l'avertissement de sécurité (le certificat est auto-signé).

<img width="1906" height="759" alt="image" src="https://github.com/user-attachments/assets/49b22cb7-954b-4707-9616-9620ad569da6" />


Le nom d'utilisateur est `admin`. Pour obtenir le mot de passe initial :

```bash
# Ce mot de passe est stocké dans un secret Kubernetes
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

<img width="1226" height="48" alt="image" src="https://github.com/user-attachments/assets/589a462b-5d3b-4c80-be01-1d70f4deea01" />

Connectez-vous. Vous êtes sur le tableau de bord d'ArgoCD !

<img width="1911" height="778" alt="image" src="https://github.com/user-attachments/assets/33bb4545-1e3e-4163-b3c7-8a9bf0d6fbf3" />


## 4. Atelier 2 : Déploiement d'Applications 

ArgoCD est extrêmement flexible. Il peut déployer des applications à partir de différentes sources de manifestes. Dans cet atelier, nous allons explorer les scénarios courants qui illustrent cette flexibilité à savoir helm et les manifest yaml.

### Cas 1 : Déploiement d'Odoo 18 avec GitOps

**Le Contexte : Un Scénario d'Entreprise**

Imaginons que nous travaillons pour une entreprise qui utilise **Odoo**, un Progiciel de Gestion Intégré (ERP) très complet. Odoo est une application critique : il gère la comptabilité, les ventes, les stocks, les ressources humaines, etc. La fiabilité et la stabilité de son déploiement sont donc primordiales.

Notre mission, en tant qu'équipe DevOps, est d'automatiser le déploiement d'Odoo sur Kubernetes de manière robuste, auditable et reproductible.

**La Solution : GitOps avec ArgoCD**

Pour ce faire, nous adoptons une approche GitOps. Au lieu de nous connecter manuellement au cluster pour y appliquer des commandes, nous décrivons l'état désiré de notre application dans un dépôt Git. ArgoCD se charge ensuite de faire de cette description une réalité dans le cluster.

Les avantages sont immenses :
- **Fiabilité :** Git devient la source unique de vérité. Fini les "ça marche sur ma machine" ou les configurations qui dérivent après des interventions manuelles.
- **Automatisation :** Toute modification de la charte Helm (mise à jour d'Odoo, changement de configuration) poussée sur Git est automatiquement déployée.
- **Audit et Sécurité :** L'historique de Git nous dit exactement qui a changé quoi et quand. Les développeurs n'ont plus besoin d'accès directs au cluster de production.
- **Rollbacks Simplifiés :** Un problème en production ? Un simple `git revert` permet de revenir à la version stable précédente.

Pour cet atelier, notre entreprise a déjà "packagé" Odoo dans sa propre charte Helm, qui se trouve ici : `https://github.com/nzapanarcisse/datascientest-chart.git`.

**Déployons Odoo avec ArgoCD**

1.  **Cliquez sur `+ NEW APP`** sur l'interface d'ArgoCD.
2.  **Remplissez les informations générales :**
    *   **Application Name** : `odoo` (Un nom simple pour identifier notre application).
    *   **Project Name** : `default` (Nous utilisons le projet par défaut d'ArgoCD).

3.  **Configurez la politique de synchronisation (Sync Policy) :**
    *   **Sync Policy** : `Automatic`
    *   Cette option est le cœur de l'automatisation. Elle indique à ArgoCD de ne pas se contenter de détecter les différences entre Git et le cluster, mais de les corriger **automatiquement**.
    *   **Cochez `Prune Resources`** : Imaginez que vous supprimez une ressource (un Service, un ConfigMap) de votre charte Helm. Sans cette option, la ressource continuerait d'exister dans le cluster. `Prune` s'assure qu'ArgoCD supprime les ressources qui n'existent plus dans Git, évitant ainsi les "ressources orphelines".
    *   **Cochez `Self Heal`** : C'est la garantie anti-dérive. Si un administrateur (ou vous-même par erreur) modifie une ressource en direct avec `kubectl` (par exemple, changer le nombre de réplicas), ArgoCD le détectera comme une déviation par rapport à la source de vérité (Git) et **annulera automatiquement** ce changement. Le cluster "s'auto-guérira" pour revenir à l'état décrit dans Git.

<img width="1915" height="549" alt="image" src="https://github.com/user-attachments/assets/3f3f9a52-65b4-48ed-9183-60bf5ba91abc" />

4.  **Définissez la Source de l'application :**
    *   **Repository URL** : `https://github.com/nzapanarcisse/datascientest-chart.git` (L'URL de notre charte Helm d'entreprise).
    *   **Revision** : `HEAD` (Indique à ArgoCD de toujours suivre la dernière version de la branche par défaut).
    *   **Path** : `odoo` (Le dossier à l'intérieur du dépôt Git qui contient les fichiers de la charte Helm).

5.  **Choisissez la Destination du déploiement :**
    *   **Cluster URL** : `https://kubernetes.default.svc` (L'alias pour le cluster sur lequel ArgoCD est lui-même installé).
    *   **Namespace** : `odoo` (Nous déploierons Odoo dans son propre espace de noms pour l'isoler des autres applications).

<img width="1884" height="813" alt="image" src="https://github.com/user-attachments/assets/73cb0430-b037-4235-b4f4-650d76bb1c35" />

6.  **Paramètres de la Charte Helm :**
    *   Dans la section **Helm**, cliquez sur `Parameters`. Nous devons indiquer à notre charte de créer le namespace pour nous (cette option est spécifique à la charte Bitnami que nous utilisons comme base).
    *   Ajoutez un paramètre : **Name**: `namespace.create`, **Value**: `true`.

7.  **Cliquez sur `CREATE`** en haut de la page.

<img width="1905" height="889" alt="image" src="https://github.com/user-attachments/assets/12dc198f-34f8-4c87-a29c-4a7fcb078c3c" />


**Que se passe-t-il ensuite ?**

ArgoCD va d'abord cloner le dépôt Git, puis comparer l'état de la charte Helm avec ce qui existe dans le namespace `odoo` du cluster. Il verra que rien n'existe et marquera l'application comme `OutOfSync`. Puisque nous avons choisi une politique `Automatic`, il va immédiatement commencer le déploiement.

Observez l'application passer par les états `Progressing` puis `Healthy` et `Synced`. Vous pouvez cliquer sur l'application pour explorer toutes les ressources Kubernetes (Deployments, Services, PersistentVolumeClaims, etc.) qui ont été créées, le tout sans une seule ligne de `kubectl` !

<img width="1299" height="508" alt="image" src="https://github.com/user-attachments/assets/fb66e28f-ccff-473a-830c-4d0ed67ea85f" />


En cliquant sur votre application Odoo depuis le tableau de bord Argo, vous pouvez voir les ressources en cours de création sur votre cluster.

<img width="1296" height="609" alt="image" src="https://github.com/user-attachments/assets/a054e4d9-2254-4fb0-9a29-e5c28db6feef" />

Vous pouvez aussi vous connecter à votre cluster et vérifier la création de ces ressources.

<img width="1285" height="230" alt="image" src="https://github.com/user-attachments/assets/de614c8c-c09f-4ea2-bb7c-c9fb5c31b0a2" />

Félicitations pour votre succès ! À ce stade, chaque fois qu'une modification sera apportée au dépôt datascientest-chart, Argo CD détectera ces changements et les appliquera sur le cluster. De plus, si un développeur ou un administrateur tente de modifier une ressource directement sur le cluster, Argo CD annulera cette modification.

<img width="1914" height="487" alt="image" src="https://github.com/user-attachments/assets/9f1a4096-af81-4572-bbfd-8910c43a331f" />


<img width="1306" height="596" alt="image" src="https://github.com/user-attachments/assets/943c0e7e-a957-471b-bdd7-55de235ed6cb" />

Ouvrez votre navigateur et allez sur `[http://VOTRE_IP_PUBLIC:30089](https://54.234.45.177:30089)`(vérifiz que ce port est bien ouvert sur votre groupe de sécurité)

<img width="1920" height="777" alt="image" src="https://github.com/user-attachments/assets/c558e8f7-c4d9-4126-a24c-ed3249d423da" />

Récupérez le mot de passe d'Odoo à partir du secret Odoo créé par la stack en exécutant la commande suivante :
```bash
kubectl -n odoo get secret odoo -o jsonpath="{.data.odoo-password}" | base64 -d
```
<img width="1892" height="37" alt="image" src="https://github.com/user-attachments/assets/3651a9a7-d13c-4ff2-9514-3da18fe02cfa" />

user:user@example.com
dz6UiV0NUR

<img width="1913" height="996" alt="image" src="https://github.com/user-attachments/assets/b2b4afbc-e8de-4410-8331-4ef882468126" />

### Cas 2 : Déploiement d'une Application Web (Chart Helm Interne)

**Le Contexte : Standardisation des Déploiements**

Continuons notre scénario. L'entreprise, satisfaite du déploiement d'Odoo, souhaite maintenant déployer sa propre application web. L'équipe "Plateforme" (dont nous faisons partie) veut standardiser les déploiements. Elle impose l'utilisation de **Helm** pour toutes les nouvelles applications, car cela permet de créer des "modèles" de déploiement réutilisables et de gérer la configuration de manière centralisée.

Notre mission est de packager l'application web (dont le code source se trouve dans le dossier `app/` de ce projet) dans une charte Helm et de la déployer avec ArgoCD.

**La Solution : Une Charte Helm pour notre App**

Nous avons déjà créé une charte Helm pour cette application et l'avons placée dans le même dépôt Git que la charte Odoo, mais dans un autre dossier.

*   **Source de la charte :** `https://github.com/nzapanarcisse/datascientest-chart.git`
*   **Chemin dans le dépôt :** `webapp/webapp-chart`

**Déployons notre application web avec sa charte Helm :**

1.  **Cliquez sur `+ NEW APP`**.
2.  **Informations Générales :**
    *   **Application Name** : `webapp`
    *   **Project Name** : `default`
3.  **Politique de Synchronisation :** `Automatic`, avec `Prune Resources` et `Self Heal`.
4.  **Source :**
    *   **Repository URL** : `https://github.com/nzapanarcisse/datascientest-chart.git` (le même dépôt que pour Odoo).
    *   **Revision** : `HEAD`
    *   **Path** : `webapp/webapp-chart` (le chemin vers notre nouvelle charte).
5.  **Destination :**
    *   **Cluster URL** : `https://kubernetes.default.svc`
    *   **Namespace** : `webapp`
6.  **Cliquez sur `CREATE`**.

ArgoCD va maintenant déployer notre application web en utilisant la charte Helm que nous avons définie. Cela montre comment une entreprise peut gérer ses propres applications packagées de manière standardisée.

<img width="1915" height="1007" alt="image" src="https://github.com/user-attachments/assets/2d4614a7-9a7b-40be-a736-09b50f3a8e9e" />

<img width="1910" height="903" alt="image" src="https://github.com/user-attachments/assets/1b548df3-f19a-4215-9b84-8be7c3f2f545" />

Vérification création de l'application sur le cluster :

<img width="1900" height="270" alt="image" src="https://github.com/user-attachments/assets/483f1d6a-d988-4a5f-bf1b-5618d158bfbf" />

Ouvrez le navigateur et allez sur `[http://VOTRE_IP_PUBLIC:30000](https://54.234.45.177:30000)`(vérifiz que ce port est bien ouvert sur votre groupe de sécurité)

<img width="1908" height="954" alt="image" src="https://github.com/user-attachments/assets/49a8ec38-dada-49f8-81ad-c191aa51eef1" />


### Cas 3 : Déploiement d'une Application Web (Manifestes Kubernetes Bruts)

**Le Contexte : Flexibilité pour les Équipes**

Un mois plus tard, une nouvelle équipe de développement rejoint l'entreprise. Ils ne sont pas encore formés à Helm et trouvent sa syntaxe complexe. Pour leur projet, ils préfèrent utiliser des **manifestes Kubernetes de base** (`deployment.yaml`, `service.yaml`), car c'est un format qu'ils maîtrisent déjà.

Notre rôle, en tant qu'équipe DevOps, est de leur montrer qu'ils peuvent tout de même bénéficier des avantages du GitOps et d'ArgoCD, même sans utiliser Helm.

**La Solution : Le GitOps avec des Manifestes Simples**

Nous avons pris les mêmes composants (un déploiement et un service) et les avons définis dans des fichiers YAML standards. Nous les avons placés dans le même dépôt central, mais dans un chemin différent.

*   **Source des manifestes :** `https://github.com/nzapanarcisse/datascientest-chart.git`
*   **Chemin dans le dépôt :** `webapp/webapp-manifest`

**Déployons la même application avec des manifestes bruts :**

1.  **Cliquez sur `+ NEW APP`**.
2.  **Informations Générales :**
    *   **Application Name** : `webapp-manifest`
    *   **Project Name** : `default`
3.  **Politique de Synchronisation :** `Automatic`, avec `Prune Resources` et `Self Heal`.
4.  **Source :**
    *   **Repository URL** : `https://github.com/nzapanarcisse/datascientest-chart.git`
    *   **Revision** : `HEAD`
    *   **Path** : `webapp/webapp-manifest` (le chemin vers nos fichiers YAML).
5.  **Destination :**
    *   **Cluster URL** : `https://kubernetes.default.svc`
    *   **Namespace** : `webapp-manifest` (un namespace différent pour éviter les conflits).
6.  **Cliquez sur `CREATE`**.

ArgoCD va lire les fichiers `deployment.yaml` et `service.yaml` et les appliquer au cluster. Le résultat final est le même (l'application est déployée), mais nous avons utilisé une méthode différente. Cela prouve aux équipes qu'ArgoCD s'adapte à leurs compétences tout en garantissant une gestion centralisée et basée sur Git.

<img width="1892" height="991" alt="image" src="https://github.com/user-attachments/assets/52b48e4f-2e6b-41f9-aaaa-860a6a3f231b" />


<img width="1891" height="1032" alt="image" src="https://github.com/user-attachments/assets/792b843e-9fa2-490e-b791-0a238750e1ef" />

Vérification création de l'application sur le cluster :

<img width="1918" height="260" alt="image" src="https://github.com/user-attachments/assets/f5b85862-0c14-4e46-a2d7-94205abe8baa" />


Ouvrez le navigateur et allez sur `[http://VOTRE_IP_PUBLIC:30001](https://54.234.45.177:30001)`(vérifiz que ce port est bien ouvert sur votre groupe de sécurité)

<img width="1882" height="915" alt="image" src="https://github.com/user-attachments/assets/19a62dc2-1e08-4e0b-8d9f-14601191796d" />


## 5. Atelier 3 : Automatisation Complète avec un Pipeline CI/CD GitOps

**Le Contexte : Passer à la Vitesse Supérieure**

Félicitations ! Dans l'atelier précédent, nous avons déployé notre application web de deux manières différentes. C'était une étape cruciale pour comprendre le fonctionnement d'ArgoCD. Cependant, le processus était encore manuel. Un développeur modifiait le code, puis un membre de l'équipe DevOps devait manuellement créer une nouvelle image, la pousser, et mettre à jour la configuration dans le dépôt GitOps.

Ce processus est lent, source d'erreurs et ne correspond pas à la vélocité attendue d'une équipe moderne. La direction souhaite maintenant une **automatisation de bout en bout**. L'objectif est simple : **lorsqu'un développeur pousse une modification du code de l'application, celle-ci doit être testée, sécurisée et déployée en production en quelques minutes, sans aucune intervention manuelle.**

C'est ici que la magie du CI/CD couplé au GitOps opère. Nous allons construire un pipeline avec GitHub Actions qui orchestrera tout ce processus.

**La Stratégie : Deux Dépôts pour Deux Responsabilités**

Pour un pipeline GitOps robuste, la meilleure pratique est de séparer les préoccupations en utilisant deux dépôts Git distincts :

1.  **Dépôt Applicatif (`argocd-datascientest`) :** C'est là que vit le code source de notre application web (le dossier `app/`). C'est le terrain de jeu des développeurs. Un `git push` sur ce dépôt déclenchera notre pipeline d'Intégration Continue (CI).

2.  **Dépôt de Configuration/GitOps (`datascientest-chart`) :** Il contient la "vérité" sur l'état de notre infrastructure (nos charts Helm, nos manifestes). C'est le dépôt surveillé par ArgoCD. Seul notre pipeline CI/CD a le droit d'écrire dedans.

![CI/CD GitOps Flow](https://i.imgur.com/gC0m3fC.png)

Ce découplage est fondamental : les développeurs n'ont pas besoin de savoir comment leur code est déployé, et l'équipe Ops garde le contrôle total sur la configuration des déploiements.

### Mise en place du pipeline GitHub Actions

Nous allons maintenant construire le workflow qui réalise ce scénario. Ce pipeline sera déclenché par un push sur le dossier `app/` de notre dépôt applicatif.

**1. Création du Fichier de Workflow**

Dans votre projet `argocd-datascientest`, créez le fichier `.github/workflows/cicd-pipeline.yml` pour le pipeline.

**2. Le Code du Workflow (`.github/workflows/cicd-pipeline.yml`)**

Copiez le code suivant dans votre fichier `.github/workflows/cicd-pipeline.yml`  ou forké le dépot pour avoir le projet sur votre propres repository.

```yaml
# =========================================================================================
# ==              PIPELINE DE CI/CD POUR L'APPLICATION WEB                             ==
# =========================================================================================
# == Ce workflow GitHub Actions est conçu pour automatiser le cycle de vie de          ==
# == notre application web, de la construction à la synchronisation avec ArgoCD.       ==
# =========================================================================================

name: CI/CD - Pipeline de Déploiement de l'Application Web

# =========================================================================================
# ==                                     TRIGGERS                                        ==
# =========================================================================================
# == Le workflow se déclenche automatiquement dans les conditions suivantes :            ==
# ==  - Un `push` est effectué sur la branche `main`.                                   ==
# ==  - Les modifications concernent le répertoire `app/`, où se trouve le code source. ==
# =========================================================================================
on:
  push:
    branches: [ "master" ]
    #paths:
    #  - 'app/**'

# =========================================================================================
# ==                                      JOBS                                         ==
# =========================================================================================
# == Le workflow est divisé en plusieurs jobs qui s'exécutent séquentiellement.        ==
# == Chaque job est une étape logique de notre pipeline : Build > Test > Scan > Push.    ==
# =========================================================================================
jobs:
  # =======================================================================================
  # == JOB 1: BUILD - Construction de l'image Docker                                    ==
  # =======================================================================================
  # == Objectif : Construire l'image Docker de l'application et la stocker en tant      ==
  # == qu'artefact pour les jobs suivants.                                               ==
  # =======================================================================================
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.generate_tag.outputs.tag }}
    steps:
      # --- Étape 1.1 : Récupération du code ---
      - name: 1.1. Récupération du code source
        uses: actions/checkout@v4

      # --- Étape 1.2 : Génération d'un tag unique ---
      - name: 1.2. Génération d'un tag unique pour l'image
        id: generate_tag
        run: echo "tag=$(echo $GITHUB_SHA | head -c7)" >> $GITHUB_OUTPUT

      # --- Étape 1.3 : Configuration de Docker Buildx ---
      - name: 1.3. Configuration de Docker Buildx
        uses: docker/setup-buildx-action@v3

      # --- Étape 1.4 : Build de l'image Docker ---
      - name: 1.4. Build de l'image Docker
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./app/Dockerfile
          push: false
          tags: datascientestuser/webapp:${{ steps.generate_tag.outputs.tag }}
          load: true

      # --- Étape 1.5 : Sauvegarde de l'image en tant qu'artefact ---
      - name: 1.5. Sauvegarde de l'image Docker pour les jobs suivants
        run: docker save datascientestuser/webapp:${{ steps.generate_tag.outputs.tag }} -o image.tar
      - uses: actions/upload-artifact@v4
        with:
          name: docker-image
          path: image.tar

  # =======================================================================================
  # == JOB 2: TEST - Test de fumée (Smoke Test) de l'application                       ==
  # =======================================================================================
  # == Objectif : Démarrer le conteneur et vérifier qu'il répond correctement.         ==
  # =======================================================================================
  test-acceptance:
    runs-on: ubuntu-latest
    needs: build
    steps:
      # --- Étape 2.1 : Récupération de l'artefact de l'image ---
      - name: 2.1. Récupération de l'image Docker
        uses: actions/download-artifact@v4
        with:
          name: docker-image

      # --- Étape 2.2 : Chargement de l'image ---
      - name: 2.2. Chargement de l'image Docker
        run: docker load -i image.tar

      # --- Étape 2.3 : Lancement du conteneur ---
      - name: 2.3. Lancement du conteneur pour le test
        run: |
          docker run -d -p 8080:80 --name webapp-test datascientestuser/webapp:${{ needs.build.outputs.image_tag }}
          echo "IMAGE_TAG=${{ needs.build.outputs.image_tag }}" >> $GITHUB_ENV

      # --- Étape 2.4 : Test de l'application ---
      - name: 2.4. Test de la réponse de l'application
        run: |
          echo "Attente du démarrage de l'application..."
          sleep 10
          echo "Vérification du contenu de la page d'accueil..."
          curl -s http://localhost:8080 | grep "Dimension"

      # --- Étape 2.5 : Nettoyage ---
      - name: 2.5. Arrêt et suppression du conteneur de test
        if: always() # S'exécute toujours, même si l'étape de test a échoué
        run: |
          docker stop webapp-test
          docker rm webapp-test

  # =======================================================================================
  # == JOB 3: SCAN - Analyse de vulnérabilités de l'image                               ==
  # =======================================================================================
  # == Objectif : Scanner l'image pour détecter des vulnérabilités de sécurité.         ==
  # =======================================================================================
  test-qualite:
    runs-on: ubuntu-latest
    needs: [build, test-acceptance]
    steps:
      - name: 3.1. Récupération de l'image Docker
        uses: actions/download-artifact@v4
        with:
          name: docker-image
      - name: 3.2. Chargement de l'image Docker
        run: |
          docker load -i image.tar
          echo "IMAGE_TAG=${{ needs.build.outputs.image_tag }}" >> $GITHUB_ENV

      - name: 3.3. Scan de vulnérabilités avec Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'datascientestuser/webapp:${{ env.IMAGE_TAG }}'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'

  # =======================================================================================
  # == JOB 4: PUSH - Publication de l'image Docker                                      ==
  # =======================================================================================
  # == Objectif : Pousser l'image, si elle est sûre, vers le registre Docker Hub.       ==
  # =======================================================================================
  push:
    runs-on: ubuntu-latest
    needs: [build, test-qualite]
    steps:
      - name: 4.1. Récupération de l'image Docker
        uses: actions/download-artifact@v4
        with:
          name: docker-image
      - name: 4.2. Chargement de l'image Docker
        run: |
          docker load -i image.tar
          echo "IMAGE_TAG=${{ needs.build.outputs.image_tag }}" >> $GITHUB_ENV

      - name: 4.3. Connexion à Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: 4.4. Push de l'image sur Docker Hub
        run: docker push datascientestuser/webapp:${{ env.IMAGE_TAG }}

  # =======================================================================================
  # == JOB 5: DEPLOY - Déclenchement de la synchronisation ArgoCD                        ==
  # =======================================================================================
  # == Objectif : Mettre à jour le dépôt GitOps pour qu'ArgoCD déploie la nouvelle image. ==
  # =======================================================================================
  trigger-argocd-sync:
    runs-on: ubuntu-latest
    needs: [build, push]
    steps:
      - name: 5.1. Checkout du dépôt de Configuration (GitOps)
        uses: actions/checkout@v4
        with:
          repository: nzapanarcisse/datascientest-chart
          token: ${{ secrets.GITOPS_PAT }}

      - name: 5.2. Mise à jour du tag de l'image dans le chart Helm
        run: |
          sed -i "s/^  tag: .*/  tag: ${{needs.build.outputs.image_tag}}/" webapp/webapp-chart/values.yaml

      - name: 5.3. Commit et Push des changements vers le dépôt GitOps
        run: |
          git config --global user.name 'GitHub Actions Bot'
          git config --global user.email 'actions@github.com'
          git add webapp/webapp-chart/values.yaml
          git diff --staged --quiet || git commit -m "CI: Mise à jour de l'image webapp vers le tag ${{needs.build.outputs.image_tag}}"
          git push
```

**3. Configuration des Secrets**

Ce pipeline a besoin de trois secrets pour fonctionner. Allez dans votre dépôt `argocd-datascientest`, puis dans `Settings > Secrets and variables > Actions` et ajoutez les secrets suivants :

*   `DOCKER_USERNAME`: `datascientestuser` ou username votre compte docker
*   `DOCKER_PASSWORD`: `datascientestuser` ou password de votre compte docker
*   `GITOPS_PAT`: Il s'agit d'un **Personal Access Token (PAT)** de GitHub. C'est crucial.
    *   Allez dans les [paramètres de votre compte GitHub](https://github.com/settings/tokens) (profile > setting > devoloper setting > `Personal access tokens` > `Tokens (classic)`.)
    *   Générez un nouveau token.
    *   Donnez-lui un nom (ex: `argocd-trigger`).
    *   **Cochez la case `repo`** pour lui donner les droits de lire et écrire dans vos dépôts.
    *   Copiez ce token et collez-le comme valeur pour le secret `GITOPS_PAT`.

<img width="1899" height="692" alt="image" src="https://github.com/user-attachments/assets/295f34d1-49c5-47a5-b9eb-db85b513d93f" />

<img width="1920" height="979" alt="image" src="https://github.com/user-attachments/assets/82943b16-f376-439f-84ea-56e292272cf9" />

### Démonstration du Workflow Automatisé

Maintenant, il est temps de voir la magie opérer. Le test ultime !

1.  **Assurez-vous que votre application `webapp` dans ArgoCD est bien configurée** pour pointer vers le chemin `webapp/webapp-chart` du dépôt `datascientest-chart` et que la synchronisation automatique est activée.

2.  **Faites une modification visible dans le code de l'application.**
    Ouvrez le fichier `app/static-website-example/index.html` et changez le titre `<h1>`.
    Par exemple : `<h1>Mon App est maintenant automatisée !</h1>`

3.  **Poussez le changement sur GitHub.**
    ```bash
    git add app/static-website-example/index.html
    git commit -m "feat: Mise à jour du titre de la page d'accueil"
    git push origin main
    ```

4.  **Observez le spectacle !**
    *   **Étape 1 (GitHub Actions) :** Allez dans l'onglet "Actions" de votre dépôt `argocd-datascientest`. Vous verrez votre pipeline se déclencher. Suivez les étapes : build, test-acceptance,test-qualité, push, et surtout le dernier job `trigger-argocd-sync` qui va commiter dans l'autre dépôt.

      <img width="1928" height="701" alt="image" src="https://github.com/user-attachments/assets/850f546d-25dc-4508-b737-da3eeb135abf" />

    *   **Étape 2 (Dépôt GitOps) :** Une fois le pipeline terminé, allez sur votre dépôt `datascientest-chart`. Vous verrez un nouveau commit fait par "GitHub Actions Bot" qui a modifié le tag de l'image dans `webapp/webapp-chart/values.yaml`.
      
      <img width="1877" height="578" alt="image" src="https://github.com/user-attachments/assets/817e163d-b6fc-47ca-851a-10d3f4c9ac29" />

    *   **Étape 3 (ArgoCD) :** Allez sur votre interface ArgoCD. Cliquez sur le bouton `Refresh` de l'application `webapp-helm`. ArgoCD va détecter que l'état désiré dans Git a changé (le tag de l'image n'est plus le même). L'application va passer à l'état `OutOfSync`.
    *   **Étape 4 (Déploiement) :** Puisque la politique de synchronisation est `Automatic`, ArgoCD va immédiatement commencer le déploiement. Il va créer un nouveau ReplicaSet, démarrer de nouveaux pods avec la nouvelle image, et une fois qu'ils sont prêts, il va supprimer les anciens. C'est un déploiement `RollingUpdate` sans interruption de service.

5.  **Vérifiez le résultat.**
    Une fois l'application `Synced` et `Healthy` dans ArgoCD, rafraîchissez la page de votre application web dans votre navigateur. Vous devriez voir le nouveau titre que vous avez modifié !

   <img width="1875" height="967" alt="image" src="https://github.com/user-attachments/assets/86dd3e47-fa7d-4577-a1ec-8707d1d42aa7" />

Vous venez de mettre en place un workflow CI/CD GitOps complet, sécurisé et entièrement automatisé. C'est le standard de l'industrie et une compétence extrêmement recherchée.

## 6. Concepts Avancés

### ArgoCD et le Multi-Cluster

**Le Contexte : L'Expansion de l'Entreprise**

Notre entreprise connaît une croissance rapide. Le déploiement de toutes les applications (développement, staging, production) sur un unique cluster Kubernetes n'est plus tenable. C'est un risque pour la stabilité : une mauvaise manipulation dans un environnement de test pourrait impacter la production. De plus, les équipes de développement ont besoin d'un environnement de `staging` stable et isolé pour valider leurs fonctionnalités avant le lancement final.

**Notre Mission :** Mettre en place une architecture multi-environnements (`production` et `staging`) gérée de manière centralisée, tout en respectant notre méthodologie GitOps.

**La Solution : Le Contrôle Multi-Cluster d'ArgoCD**

ArgoCD a été conçu pour cela. Une seule instance d'ArgoCD, tournant sur notre cluster principal (que nous appellerons désormais le cluster `prod`), peut gérer des déploiements sur un nombre illimité de clusters Kubernetes distants.

Nous allons enregistrer notre nouveau cluster `staging` auprès de notre instance ArgoCD centrale. Cela nous permettra de :
-   **Garder un point de contrôle unique :** Toute la gestion des déploiements se fait depuis une seule interface.
-   **Appliquer le GitOps partout :** La source de vérité reste Git, que l'on déploie en `staging` ou en `prod`.
-   **Isoler les environnements :** Les clusters sont indépendants, renforçant la sécurité et la stabilité.

#### Étape 1 : Enregistrer un Cluster Externe

Supposons que vous ayez déjà configuré l'accès à votre nouveau cluster `staging` et que son contexte (`staging-context`) soit présent dans votre fichier `~/.kube/config` local.

1.  **Installez la CLI d'ArgoCD** sur votre machine locale si ce n'est pas déjà fait.

2.  **Connectez-vous à votre instance ArgoCD :**
    ```bash
    # Remplacez l'IP par celle de votre serveur ArgoCD
    argocd login <ARGO_CD_SERVER_IP>
    ```

3.  **Listez les contextes Kubernetes connus par votre machine locale :**
    ```bash
    kubectl config get-contexts
    # Vous devriez voir le contexte de votre cluster de production et celui de staging.
    ```

4.  **Enregistrez le nouveau cluster `staging` auprès d'ArgoCD :**
    ```bash
    # Remplacez <STAGING_CONTEXT_NAME> par le nom du contexte de votre cluster de staging
    argocd cluster add <STAGING_CONTEXT_NAME>
    ```
    Cette commande est puissante. ArgoCD va se connecter au cluster distant, y créer un `ServiceAccount`, un `ClusterRole` et un `ClusterRoleBinding`. Cela lui donne les permissions nécessaires pour gérer les ressources sur ce cluster distant, sans jamais exposer les credentials du cluster `staging` à l'extérieur.

#### Étape 2 : Déployer une Application sur le Cluster de Staging

Maintenant que le cluster est enregistré, déployer dessus est un jeu d'enfant. Nous allons déployer notre application `webapp` sur ce nouvel environnement.

La meilleure pratique GitOps est de définir l'application de manière déclarative. Créons un fichier `webapp-staging-app.yaml` :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  # Un nom qui identifie clairement l'application et son environnement
  name: webapp-staging
  namespace: argocd
spec:
  project: default
  source:
    # Le même dépôt Git que pour la production
    repoURL: 'https://github.com/nzapanarcisse/datascientest-chart.git'
    path: webapp/webapp-chart
    targetRevision: HEAD # On peut aussi pointer vers une branche "staging"
  destination:
    # On cible le cluster distant !
    # Le nom est celui retourné par la commande `argocd cluster list`
    name: <NOM_DU_CLUSTER_STAGING>
    # On déploie dans un namespace dédié sur le cluster de staging
    namespace: webapp-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    # Important : On demande à ArgoCD de créer le namespace s'il n'existe pas
    syncOptions:
    - CreateNamespace=true
```

Appliquez ce manifeste sur votre cluster principal (celui où tourne ArgoCD) :
```bash
kubectl apply -f webapp-staging-app.yaml -n argocd
```

Et voilà ! Rendez-vous sur l'interface d'ArgoCD. Vous verrez une nouvelle application `webapp-staging`. En cliquant dessus, vous constaterez qu'elle déploie ses ressources sur le cluster distant, prouvant que vous maîtrisez désormais une architecture multi-environnements entièrement pilotée par Git.

### Notifications avec Slack

**Le Contexte : Améliorer la Visibilité et la Réactivité**

Notre pipeline CI/CD est automatisé, et ArgoCD garantit que nos clusters reflètent l'état de Git. C'est excellent, mais l'équipe fonctionne encore en mode réactif. Pour savoir si un déploiement a réussi ou échoué, un développeur doit aller sur l'interface d'ArgoCD. Si quelque chose tourne mal, le temps de détection dépend de la vigilance manuelle de l'équipe.

**Notre Mission :** Mettre en place un système de notifications proactives. L'équipe doit être informée en temps réel des événements importants du cycle de vie de nos applications (déploiement réussi, échec, dégradation de la santé) directement dans son outil de communication principal : **Slack**.

**La Solution : L'Intégration `ArgoCD Notifications`**

ArgoCD dispose d'un sous-système puissant dédié aux notifications. Il peut s'intégrer à des dizaines d'outils (Slack, Microsoft Teams, email, etc.). Nous allons le configurer pour qu'il envoie des messages clairs et contextuels à un canal Slack dédié.

Cela va créer une boucle de feedback instantanée, permettant à l'équipe de :
-   **Célébrer les déploiements réussis.**
-   **Réagir immédiatement en cas d'échec.**
-   **Maintenir un journal d'audit visible par tous.**

#### Étape 1 : Créer un Webhook sur Slack

1.  Allez sur `api.slack.com/apps`.
2.  Créez une nouvelle application pour votre workspace.
3.  Dans le menu `Features`, activez **Incoming Webhooks**.
4.  Cliquez sur **Add New Webhook to Workspace**, choisissez un canal (ex: `#deployments`) et autorisez.
5.  Copiez l'URL du webhook. Elle ressemble à `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX`.

#### Étape 2 : Configurer ArgoCD Notifications

Nous devons stocker cette URL de webhook de manière sécurisée dans un Secret Kubernetes et informer le contrôleur de notifications comment l'utiliser.

1.  **Créez le Secret contenant l'URL du webhook :**
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Secret
    metadata:
      name: argocd-notifications-secret
      namespace: argocd
    stringData:
      # La clé 'slack-url' est arbitraire, mais nous la réutiliserons
      slack-url: <VOTRE_URL_DE_WEBHOOK_SLACK>
    EOF
    ```

2.  **Modifiez le ConfigMap `argocd-notifications-cm` pour définir le service Slack :**
    Ce ConfigMap est le cerveau des notifications. Nous y définissons les "services" (comment se connecter à Slack) et les "templates" (à quoi ressembleront les messages).
    ```bash
    kubectl patch configmap argocd-notifications-cm -n argocd --type merge -p '{"data": {"service.slack": "url: $slack-url", "context": "slack-url: $slack-url"}}'
    ```
    *Note : Nous ajoutons `slack-url` au `context` pour le rendre disponible dans les templates de message.*

#### Étape 3 : Créer un Template de Message (Optionnel mais recommandé)

Pour des messages plus riches, définissons un template.
```bash
kubectl patch configmap argocd-notifications-cm -n argocd --type merge -p '{"data": {"template.on-sync-succeeded": "message: |\n  L''application {{.app.metadata.name}} a été synchronisée avec succès.\n  Commit: `{{.app.status.sync.revision}}`\n  Auteur: {{.app.revisionMetadata.author}}\n  Consultez l''application ici: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}"}}'
```

#### Étape 4 : S'abonner aux Notifications

Maintenant, il suffit de "dire" à notre application qu'elle doit envoyer des notifications. Cela se fait via une simple annotation sur l'objet `Application` d'ArgoCD.

Modifions notre application `webapp` pour qu'elle notifie sur le canal `#deployments` à chaque synchronisation réussie.

```bash
# Remplacez 'webapp' par le nom de votre application et '#deployments' par votre canal
kubectl patch app webapp -n argocd -p '{"metadata": {"annotations": {"notifications.argoproj.io/subscribe.on-sync-succeeded.slack": "#deployments"}}}' --type merge
```
Cette annotation est très lisible : `subscribe` à l'événement `on-sync-succeeded` en utilisant le service `slack` et en envoyant au canal `#deployments`.

Désormais, chaque fois que l'application `webapp` sera synchronisée avec succès (après un `git push` ou une action manuelle), un message apparaîtra sur Slack, informant toute l'équipe en temps réel !

## 7. Conclusion

Félicitations ! Vous avez non seulement appris à utiliser un outil, mais vous avez adopté une **méthodologie complète** qui transforme la manière de livrer des logiciels. En parcourant ce cours, vous avez :

-   **Intériorisé les principes fondamentaux du GitOps**, en faisant de Git la source unique de vérité pour l'état de votre infrastructure.
-   **Construit un environnement de production réaliste** avec Kubernetes et ArgoCD, de l'installation à la configuration.
-   **Déployé diverses applications** en utilisant les stratégies les plus courantes (Charts Helm, manifestes bruts), prouvant la flexibilité d'ArgoCD.
-   **Orchestré un pipeline CI/CD entièrement automatisé** avec GitHub Actions, réalisant la promesse d'un déploiement fluide et sécurisé du code à la production.
-   **Exploré des concepts avancés et cruciaux** comme la gestion multi-cluster et les notifications, vous préparant à des scénarios d'entreprise complexes.

Le voyage ne s'arrête pas là. ArgoCD fait partie d'un écosystème plus large. Nous vous encourageons à explorer :
-   **Argo Rollouts :** Pour des stratégies de déploiement avancées (Canary, Blue-Green).
-   **Argo Workflows :** Pour orchestrer des tâches parallèles complexes en tant que workflows Kubernetes-natifs.
-   **Argo Events :** Pour déclencher des actions Kubernetes en réponse à des événements externes (webhooks, messages, etc.).

Vous détenez désormais les clés pour construire des plateformes de livraison continue robustes, sécurisées et véloces. Le GitOps est plus qu'une compétence technique, c'est un changement de culture, et vous êtes maintenant prêt à en être l'ambassadeur. Bon déploiement !

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
