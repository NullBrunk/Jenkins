<div align="center">

# Jenkinsfiles


</div>

| Pipeline              | Utilité          | 
| :---------------------| :--------------- | 
| <a href="https://github.com/mondecole/jenkins/tree/main/pipelines/ec2_pipeline/">EC2 Pipeline</a>        | Pipeline complète avec DevSecOps, Quality Gate et notifications |
| <a href="https://github.com/mondecole/jenkins/tree/main/pipelines/k8s_pipeline/">K8s Pipeline</a>        | Variante Kubernetes remplaçant le déploiement EC2 classique |
| <a href="https://github.com/mondecole/jenkins/tree/main/pipelines/travelops_pipeline/">Travelops Pipeline</a>  | Pipeline spécifique au projet TravelOps |



<br>


## 🔄 Base Pipeline

Toutes les pipelines ci dessous sont basés sur une Pipeline initiale qui possède les fonctionnalités suivantes:

**Fonctionnalités :**
- récupération du code depuis GitHub ;
- compilation du backend (Maven) ;
- installation et build du frontend (Node.js) ;
- exécution des tests backend ;
- exécution des tests frontend.
- build des images Docker backend/frontend ;
- push vers Docker Hub ;

#### 📈 Workflow

```
Continuous Integration
  |
  ├── GitHub (checkout)
  ├── Build Backend (Maven)
  ├── Tests Backend
  ├── Build Frontend (npm)
  ├── Tests Frontend
  |
Continuous Deployment
  |
  ├── Docker Build backend
  ├── Docker Build frontend
  └── Docker Hub (push)

```


## ⚙️ EC2 Pipeline

_Version complète avec ajout du déploiement continu sur AWS EC2, orientée production intégrant les aspects DevSecOps, qualité et observabilité._


#### 🎯 Fonctionnalités :

**Ajouts:**
- connexion SSH aux instances EC2 ;
- mise à jour automatique via : ``docker compose pull``

**Infrastructure**
- utilisation de DNS privés AWS (Route53) ;
- variables centralisées pour faciliter la maintenance ;
- déploiement backend/frontend sur EC2.

**Qualité logicielle**
- analyse de code avec SonarQube ;
- vérification de la qualité via Quality Gate.

**Sécurité**
- scan des images Docker avec Trivy ;
- génération automatique d’un rapport HTML de sécurité.

**Monitoring CI/CD**
- notifications Discord :
  - démarrage du build ;
  - statut final (succès / échec).



#### 📈 Workflow

```
Logging
  |
  ├── Discord Notification (start, end)
  |
Continuous Integration
  |
  ├── GitHub (checkout)
  ├── Build Backend (Maven)
  ├── Tests Backend
  ├── Build Frontend (npm)
  ├── Tests Frontend
  |
Security
  |
  ├── Code Analysis (sonarqube)
  ├── Security Scan (trivy)
  |
Continuous Deployment
  |
  ├── Docker Build backend
  ├── Docker Build frontend
  ├── Docker Hub (push)
  └── EC2 Deployment
```

Source: <a href="https://github.com/mondecole/jenkins/tree/main/pipelines/ec2_pipeline/">pipelines/ec2_pipeline/</a>



### ☸️ k8s Pipeline

_Version Kubernetes de la pipeline EC2._

Différence principale : le déploiement n’est plus effectué directement sur une instance EC2. Donc dans la partie CD on remplace le `EC2 Deployment` par:

- mise à jour du déploiement Kubernetes (`kubectl rollout`);
- redéploiement automatique des pods backend par le Control Plane.


#### 📈 Workflow


```
Logging
  |
  ├── Discord Notification (start, end)
  |
Continuous Integration
  |
  ├── GitHub (checkout)
  ├── Build Backend (Maven)
  ├── Tests Backend
  ├── Build Frontend (npm)
  ├── Tests Frontend
  |
Security
  |
  ├── Code Analysis (sonarqube)
  ├── Security Scan (trivy)
  |
Continuous Deployment
  |
  ├── Docker Build backend
  ├── Docker Build frontend
  ├── Docker Hub (push)
  |
Kubernetes 
  |
  ├── kubectl rollout
  └── Pod Update
```

Source: <a href="https://github.com/mondecole/jenkins/tree/main/pipelines/k8s_pipeline/">pipelines/k8s_pipeline/</a>


### TravelOps Pipeline

_Pipeline dédiée au projet TravelOps._

Fonctionnalités :
- build backend ;
- build frontend ;
- tests frontend ;
- déploiement distant sur serveur de production ;
- mise à jour automatique du dépôt ;
- redéploiement via Docker Compose ;
- notifications Discord.

#### Workflow
```

Logging
  |
  ├── Discord Notification (start, end)
  |
Continuous Integration
  |
  ├── GitHub (checkout)
  ├── Build Backend (Maven)
  ├── Build Frontend (npm)
  |
Continuous Deployment
  |
  └── EC2 Deployment (docker compose pull)
```

Source: <a href="https://github.com/mondecole/jenkins/tree/main/pipelines/travelops_pipeline/">pipelines/travelops_pipeline/</a>
