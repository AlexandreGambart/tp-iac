# TP — Infrastructure as Code avec Terraform et Docker

---

## 1. Objectifs pédagogiques

À la fin de ce TP, vous serez capable de :

- expliquer le principe de l'Infrastructure as Code (IaC) ;
- décrire des ressources avec Terraform ;
- initialiser un projet et valider sa configuration ;
- examiner un plan avant d'appliquer des changements ;
- déployer un serveur Nginx dans un conteneur Docker ;
- personnaliser le service et son réseau ;
- utiliser des variables et des outputs ;
- comprendre le rôle du fichier d'état Terraform ;
- versionner le code avec Git ;
- automatiser des contrôles de base avec GitHub Actions ;
- supprimer proprement les ressources créées.

## 2. Architecture

Le TP utilise Docker en local. Il ne nécessite donc pas de compte AWS et ne crée pas de ressources Cloud facturées.

```text
Fichiers Terraform
(main.tf, variables.tf, outputs.tf)
             |
             v
         Terraform
    init -> plan -> apply
             |
             v
       Moteur Docker
             |
             v
      Conteneur Nginx
       port interne 80
             |
             v
  http://localhost:8080
```

Terraform décrit l'état souhaité. Le provider Docker traduit cette description en opérations sur Docker.

## 3. Prérequis

Installez les outils suivants :

- [Terraform](https://developer.hashicorp.com/terraform/install)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) ou Docker Engine
- [Git](https://git-scm.com/downloads) (pour les étapes de versionnement)
- un éditeur par exemple Visual Studio Code

Vérifiez les installations :

```bash
terraform -version
docker --version
docker info
git --version
```

La commande `docker info` doit fonctionner. Si elle échoue, démarrez Docker Desktop ou le service Docker avant de continuer.

## 4. Créer la structure du projet

Créez le dossier et les fichiers suivants :

```text
tp-iac/
├── main.tf
├── variables.tf
├── outputs.tf
├── .gitignore
├── Makefile
├── dev.tfvars
├── html/
│   └── index.html
└── .github/
    └── workflows/
        └── terraform.yml
```

Les fichiers `html/index.html`, `Makefile`, `dev.tfvars` et le workflow seront créés dans les étapes suivantes.

## 5. Décrire l'infrastructure avec Terraform

### 5.1 Fichier `main.tf`

Créez `main.tf` avec le contenu suivant :

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}

resource "docker_image" "nginx" {
  name         = "nginx:alpine"
  keep_locally = true
}

resource "docker_network" "app_network" {
  name = "${var.container_name}-network"
}

resource "docker_container" "web" {
  name  = var.container_name
  image = docker_image.nginx.image_id

  ports {
    internal = 80
    external = var.external_port
    ip       = "127.0.0.1"
  }

  networks_advanced {
    name = docker_network.app_network.name
  }

  mounts {
    target = "/usr/share/nginx/html"
    source = abspath("${path.module}/html")
    type   = "bind"
  }
}
```

Ce fichier déclare :

- une image Nginx ;
- un réseau Docker dédié ;
- un conteneur Web ;
- une redirection du port local vers le port 80 du conteneur ;
- un montage du dossier local `html/` dans le répertoire Web de Nginx.

Le dossier `html/` doit exister avant le déploiement.

### 5.2 Fichier `variables.tf`

```hcl
variable "container_name" {
  description = "Nom du conteneur Web"
  type        = string
  default     = "tp-iac-nginx"

  validation {
    condition     = can(regex("^[a-zA-Z0-9][a-zA-Z0-9_.-]*$", var.container_name))
    error_message = "Le nom du conteneur contient des caractères non autorisés."
  }
}

variable "external_port" {
  description = "Port HTTP exposé sur la machine locale"
  type        = number
  default     = 8080

  validation {
    condition     = var.external_port >= 1024 && var.external_port <= 65535
    error_message = "Le port doit être compris entre 1024 et 65535."
  }
}
```

Les variables permettent de modifier les paramètres sans réécrire les ressources. La validation du port limite ici les valeurs à la plage 1024–65535.

### 5.3 Fichier `outputs.tf`

```hcl
output "container_name" {
  description = "Nom du conteneur déployé"
  value       = docker_container.web.name
}

output "network_name" {
  description = "Nom du réseau Docker"
  value       = docker_network.app_network.name
}

output "website_url" {
  description = "URL du serveur Web local"
  value       = "http://localhost:${var.external_port}"
}
```

Les outputs affichent des informations utiles après le déploiement.

### 5.4 Fichier `.gitignore`

```gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
crash.*.log
*.tfvars
*.tfplan
```

**Important :** ne publiez pas l'état Terraform ni les fichiers de variables qui pourraient contenir des données sensibles. Le fichier `.terraform.lock.hcl`, généré par Terraform, doit en revanche être conservé dans Git afin de verrouiller les versions des providers.

### 5.5 Créer la page HTML

Créez `html/index.html` :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>TP Infrastructure as Code</title>
</head>
<body>
  <h1>Infrastructure déployée avec Terraform</h1>
  <p>Serveur Nginx exécuté dans Docker.</p>
  <p>Déploiement géré par Infrastructure as Code.</p>
</body>
</html>
```

## 6. Initialiser et valider Terraform

Placez-vous dans le dossier du projet :

```bash
cd tp-iac
```

Initialisez Terraform :

```bash
terraform init
```

Formatez et validez la configuration :

```bash
terraform fmt -recursive
terraform validate
```

Résultats attendus :

- `terraform init` initialise le projet et télécharge le provider Docker ;
- `terraform fmt` met les fichiers au format standard ;
- `terraform validate` vérifie la cohérence de la configuration.

Si une erreur apparaît, corrigez-la avant de continuer.

## 7. Planifier et déployer

### 7.1 Examiner le plan

```bash
terraform plan
```

Terraform doit présenter les ressources qu'il prévoit de créer. Le plan ne déploie pas l'infrastructure.

### 7.2 Appliquer la configuration

```bash
terraform apply
```

Examinez le plan affiché puis saisissez `yes` lorsque Terraform le demande.

Terraform télécharge l'image Nginx si nécessaire, crée le réseau et démarre le conteneur.

### 7.3 Afficher les outputs

```bash
terraform output
```

L'URL attendue par défaut est `http://localhost:8080`.

## 8. Tester le serveur Web

Ouvrez l'URL suivante dans un navigateur :

<http://localhost:8080>

Vous pouvez également tester avec :

```bash
curl http://localhost:8080
docker ps
docker network ls
```

La page HTML personnalisée doit apparaître. La commande `docker ps` doit montrer le conteneur en cours d'exécution.

## 9. Personnaliser la page Web

1. Modifiez `html/index.html`.
2. Ajoutez un titre, un paragraphe et la date du TP.
3. Rechargez `http://localhost:8080`.
4. Vérifiez que la modification apparaît.

Le montage de type bind permet de partager le dossier local avec le conteneur. Une modification du fichier HTML local est donc normalement visible sans reconstruire l'image.

**Question :** quelle est la différence entre un montage bind et la copie d'un fichier dans une image Docker ?

## 10. Créer un réseau Docker dédié

La ressource `docker_network.app_network` et le bloc `networks_advanced` du conteneur sont déjà déclarés dans `main.tf`.

Inspectez le réseau :

```bash
docker network inspect tp-iac-nginx-network
```

Si vous avez modifié `container_name`, adaptez le nom du réseau en conséquence.

**Travail demandé :**

- expliquez pourquoi le conteneur est connecté à ce réseau ;
- indiquez l'intérêt de déclarer le réseau dans Terraform ;
- vérifiez que le réseau est supprimé lors de la destruction du projet.

## 11. Limiter l'exposition du service

Dans le bloc `ports`, la propriété suivante limite l'écoute au loopback de la machine :

```hcl
ip = "127.0.0.1"
```

Cela évite d'exposer directement le port du conteneur sur toutes les interfaces réseau de l'hôte.

Appliquez et testez :

```bash
terraform plan
terraform apply
curl http://127.0.0.1:8080
```

**Question :** quelle est la différence entre `127.0.0.1` et une écoute sur toutes les interfaces ?

## 12. Paramétrer les environnements avec des variables

Créez `dev.tfvars` :

```hcl
container_name = "tp-iac-dev"
external_port  = 8081
```

Déployez avec ces paramètres :

```bash
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"
```

Testez ensuite :

<http://localhost:8081>

**Attention :** les fichiers de variables ne créent pas automatiquement des états séparés. Si vous utilisez plusieurs configurations, isolez soigneusement leurs états et évitez de faire gérer la même ressource par plusieurs états Terraform.

Si vous passez d'une configuration à l'autre, vérifiez toujours le plan avant d'appliquer. Avec le même état local, Terraform peut vouloir remplacer les ressources existantes.

## 13. Inspecter l'état Terraform

Exécutez :

```bash
terraform state list
terraform state show docker_container.web
terraform output
```

Le fichier `terraform.tfstate` permet à Terraform de relier les ressources déclarées dans le code aux objets qu'il gère réellement.

**Questions :**

1. Quelles ressources apparaissent dans `terraform state list` ?
2. Quelles informations affiche `terraform state show` ?
3. Pourquoi l'état Terraform doit-il être protégé ?
4. Pourquoi faut-il éviter de publier cet état dans un dépôt public ?

## 14. Observer une dérive de configuration

Une dérive est une différence entre la configuration attendue et la réalité de l'infrastructure.

1. Arrêtez le conteneur manuellement :

   ```bash
   docker stop tp-iac-nginx
   ```

2. Examinez l'état de l'infrastructure :

   ```bash
   terraform plan
   ```

3. Analysez le plan et les différences détectées.
4. Si nécessaire, réappliquez la configuration :

   ```bash
   terraform apply
   ```

Adaptez le nom du conteneur si vous utilisez `dev.tfvars`.

Le comportement exact dépend du provider et de la ressource. Cette expérience sert à observer comment Terraform compare l'état connu, la configuration et les informations disponibles auprès du provider.

## 15. Versionner avec Git

Initialisez le dépôt :

```bash
git init
git add main.tf variables.tf outputs.tf .gitignore html/
git add .terraform.lock.hcl
git commit -m "Initialisation du projet IaC"
```

Si le fichier `.terraform.lock.hcl` n'existe pas encore, lancez `terraform init` puis recommencez l'ajout.

Après une modification :

```bash
git status
git diff
git add main.tf
git commit -m "Ajout du réseau Docker"
```

**Travail demandé :** expliquez comment l'historique Git facilite la revue, la collaboration et le retour sur une modification.

## 16. Automatiser avec un Makefile

Créez un fichier nommé `Makefile` :

```makefile
.PHONY: init fmt validate plan apply destroy

init:
	terraform init

fmt:
	terraform fmt -recursive

validate:
	terraform validate

plan:
	terraform plan

apply:
	terraform apply

destroy:
	terraform destroy
```

Les commandes sous chaque cible doivent commencer par une **tabulation**, et non des espaces.

Exécutez :

```bash
make init
make fmt
make validate
make plan
```

Le Makefile simplifie les commandes récurrentes. Il ne remplace pas la vérification du plan avant un déploiement.

## 17. Ajouter des contrôles CI avec GitHub Actions

Créez `.github/workflows/terraform.yml` :

```yaml
name: Terraform Checks

on:
  push:
  pull_request:

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Initialize
        run: terraform init -backend=false

      - name: Format Check
        run: terraform fmt -check -recursive

      - name: Validate
        run: terraform validate
```

Cette pipeline exécute des contrôles de formatage et de validation sur GitHub Actions. Elle n'effectue pas le déploiement Docker, car le runner n'est pas configuré pour piloter le moteur Docker de votre machine.

Pour un usage professionnel, vérifiez les versions des actions, appliquez une politique de mise à jour et configurez les autorisations minimales nécessaires.

## 18. Détruire l'infrastructure

Quand les tests sont terminés, examinez d'abord les ressources concernées :

```bash
terraform plan -destroy
```

Puis détruisez les ressources gérées par Terraform :

```bash
terraform destroy
```

Confirmez avec `yes`.

Vérifiez ensuite :

```bash
docker ps -a
docker network ls
terraform state list
```

Le conteneur et le réseau gérés par ce projet doivent avoir été supprimés. L'image Nginx peut rester localement, car `keep_locally = true` demande au provider de la conserver.

## 19. Exercices complémentaires

Réalisez les exercices suivants sans consulter immédiatement une correction.

### Exercice 1 — Modifier le nom

- Remplacez le nom du conteneur par `tp-iac-web`.
- Exécutez `terraform fmt`, `terraform validate` et `terraform plan`.
- Expliquez les changements prévus.

### Exercice 2 — Modifier le port

- Exposez le serveur sur le port `8082`.
- Vérifiez le plan.
- Appliquez la modification et testez `http://localhost:8082`.

### Exercice 3 — Ajouter une variable

- Ajoutez une variable pour choisir le nom du réseau.
- Utilisez-la dans la ressource `docker_network`.
- Ajoutez un output pour afficher le nom choisi.

### Exercice 4 — Comprendre l'état

- Listez les ressources Terraform.
- Identifiez les dépendances entre l'image, le réseau et le conteneur.
- Expliquez pourquoi le conteneur dépend de l'image et du réseau.

### Exercice 5 — Détecter une dérive

- Arrêtez le conteneur depuis Docker.
- Lancez `terraform plan`.
- Notez le comportement observé.
- Restaurez l'infrastructure.

### Exercice 6 — CI/CD

- Ajoutez le projet à un dépôt GitHub.
- Poussez votre code.
- Vérifiez que le workflow s'exécute.
- Provoquez une erreur de formatage, puis observez le résultat du contrôle.

### Exercice 7 — Nettoyage

- Exécutez `terraform plan -destroy`.
- Détruisez les ressources.
- Vérifiez l'absence du conteneur et du réseau.
- Expliquez pourquoi il ne faut pas supprimer manuellement l'état Terraform pour nettoyer une infrastructure active.

## 20. Questions de compréhension

1. Quelle est la différence entre `terraform init`, `terraform plan` et `terraform apply` ?
2. Quel est le rôle d'un provider Terraform ?
3. À quoi sert le fichier `terraform.tfstate` ?
4. Pourquoi utilise-t-on des variables et des outputs ?
5. Qu'est-ce qu'une dérive de configuration ?
6. Quelle différence existe-t-il entre Infrastructure as Code et configuration manuelle ?
7. Quel est l'intérêt de Git dans un projet IaC ?
8. Quelle est la différence entre un contrôle CI et un déploiement CD ?
9. Pourquoi faut-il examiner le plan avant d'appliquer une modification ?
10. Quelles précautions faut-il prendre avec les fichiers d'état et les secrets ?

## 21. Grille d'évaluation

| Compétence | Points |
|---|---:|
| Fichiers Terraform fonctionnels | 3 |
| Page Web personnalisée | 2 |
| Réseau Docker et sécurité | 3 |
| Variables et état Terraform | 3 |
| Git et automatisation | 4 |
| Tests et nettoyage | 5 |
| **Total** | **20** |

## 22. Dépannage

### Terraform ne trouve pas Docker

- Démarrez Docker Desktop ou le service Docker.
- Vérifiez `docker info`.
- Relancez `terraform plan`.

### Le port est déjà utilisé

Choisissez un autre port, par exemple `8082`, puis relancez le plan et l'application.

### Le conteneur ne démarre pas

Inspectez-le avec :

```bash
docker ps -a
docker logs tp-iac-nginx
```

Adaptez le nom si vous utilisez une autre variable.

### Terraform signale une erreur de validation

Exécutez :

```bash
terraform fmt -recursive
terraform validate
```

Lisez le fichier et la ligne mentionnés dans le message d'erreur.

### Le dossier HTML n'existe pas

Vérifiez que `html/index.html` a été créé avant le premier `terraform apply`.

---

## Livrable attendu

Remettez un dépôt Git contenant :

- les fichiers Terraform (`main.tf`, `variables.tf`, `outputs.tf`) ;
- la page HTML personnalisée ;
- le fichier `.gitignore` ;
- le `Makefile` ;
- le workflow GitHub Actions ;
- un README expliquant l'installation, le déploiement, les tests et la destruction.

**Conclusion :** ce TP met en pratique le cycle IaC : décrire, initialiser, planifier, déployer, vérifier, versionner, automatiser et détruire.
