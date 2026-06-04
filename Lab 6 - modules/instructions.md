# LAB 06 - Modularisation & "Inner Source"

**Contexte de l'Entreprise TERRA** :
L'équipe Architecture Cloud a validé votre configuration (Data Source + Lifecycle). Elle souhaite maintenant que **toutes** les équipes de la banque utilisent exactement cette configuration, sans avoir à copier-coller votre code.
Vous devez créer un **Module Terraform Standardisé** ("Black Box") qui force l'utilisation de l'image Golden et des tags obligatoires.

**Objectifs** :

1. **Producteur** : Créer un dépôt Git séparé contenant la logique de l'instance.
2. **Versioning** : Figer une version stable (`v1.0.0`) via un Tag Git.
3. **Consommateur** : Nettoyer le projet `lab01-audit` pour qu'il appelle simplement ce module.

**Environnement** :

* **Repo A (Module)** : `<votre prenom>-tf-module-aws-standard-ec2` (Nouveau)
* **Repo B (Projet)** : `lab01-audit` (Existant)

---

## Partie 1 : Création du "Producteur" (Le Module)

Nous allons extraire la complexité (`data source`, `lifecycle`, `tags logic`) dans une boîte noire.

### 1. Création du Dépôt Module

1. Sur GitLab, créez un **Nouveau Projet** (Blank project).
2. Nommez-le : `<votre prenom>-tf-module-aws-standard-ec2`.
3. **Important** : Visibilité **Publique** _(c'est pour les besoins du Lab, nous evitons les confiurations d'authentifications de GitLab)_.
4. Clonez-le sur votre machine dans un dossier séparé (pas dans `lab01-audit`).

### 2. Migration du Code (`main.tf` du module)

Dans ce nouveau dossier, créez un fichier `main.tf`. Copiez-y la logique "intelligente" de votre ancien projet, mais **rendez-la générique**.

*Note : On retire le `for_each` ici. Le module représente UNE instance. C'est le consommateur qui fera la boucle.*

```hcl
# tf-module-aws-standard-ec2/main.tf

# 1. Le Module impose la Golden Image (SecOps requirement)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# 2. La Ressource Standardisée
resource "aws_instance" "this" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type # Variable d'input
  
  # On attache les SGs fournis par le consommateur
  vpc_security_group_ids = var.security_group_ids 

  # User Data injecté depuis l'extérieur
  user_data = var.user_data_script
  user_data_replace_on_change = true

  tags = merge(var.custom_tags, {
    CreatedBy = "Terraform-Module-Standard-v1"
    GoldenImage = "True"
  })

  lifecycle {
    create_before_destroy = true
  }
}

```

### 3. Définition des Entrées (`variables.tf`)

Le module a besoin d'ingrédients pour fonctionner.

```hcl
# tf-module-aws-standard-ec2/variables.tf

variable "instance_type" {
  description = "Taille de l'instance (ex: t3.micro)"
  type        = string
  default     = "t3.micro"
}

variable "security_group_ids" {
  description = "Liste des IDs de Security Groups à attacher"
  type        = list(string)
}

variable "user_data_script" {
  description = "Contenu du script de démarrage (User Data)"
  type        = string
  default     = null
}

variable "custom_tags" {
  description = "Tags supplémentaires spécifiques au projet"
  type        = map(string)
  default     = {}
}

```

### 4. Définition des Sorties (`outputs.tf`)

Le module doit renvoyer des infos au consommateur (ex: l'IP pour l'inventaire).

```hcl
# tf-module-aws-standard-ec2/outputs.tf

output "id" {
  value = aws_instance.this.id
}

output "public_ip" {
  value = aws_instance.this.public_ip
}

```

---

## Partie 2 : Versioning (La Release)

Pour éviter de casser la prod de tout le monde si on modifie le module, on utilise des versions strictes.

**Dans le dossier du module :**

```bash
# 1. Envoyer le code
git add .
git commit -m "Initial commit of Standard EC2 Module"
git push

# 2. Créer une version figée (Tag)
git tag -a v1.0.0 -m "Release v1.0.0 - Stable"

# 3. Envoyer le tag au serveur
git push origin v1.0.0

```

*Vérification : Sur GitLab, dans le menu de gauche "Code > Tags", vous devez voir `v1.0.0`.*

---

## Partie 3 : Le "Consommateur" (Mise à jour du Lab 01)

Revenez dans votre projet principal `lab01-audit`. Nous allons supprimer du code !

### 1. Appel du Module (`main.tf`)

Ouvrez `main.tf`.

1. **Supprimez** le bloc `data "aws_ami"`. (C'est le module qui gère ça maintenant).
2. **Supprimez** le bloc `resource "aws_instance"`.
3. Remplacez-le par l'appel au module :

```hcl
# main.tf du projet Lab01

# Note : On garde le Security Group ici, car c'est spécifique au réseau du projet
resource "aws_security_group" "audit_sg" {
  # ... (votre code existant reste inchangé)
}

# APPEL DU MODULE
module "ec2_fleet" {
  # LA SOURCE GIT AVEC LE TAG v1.0.0
  # Syntaxe : git::https://gitlab.com/<GROUPE>/<PROJET>.git?ref=v1.0.0
  source = "git::https://gitlab.wizetraining.com/<VOTRE_USER>/<votre prenom>-tf-module-aws-standard-ec2.git?ref=v1.0.0"

  # La boucle for_each est déplacée sur l'appel du module !
  for_each = var.servers

  # Inputs (Variables du module)
  instance_type      = each.value.type
  security_group_ids = [aws_security_group.audit_sg.id]
  
  # On passe le script calculé localement
  user_data_script   = templatefile("${path.module}/scripts/install_nginx.tftpl", {
    env_name     = var.environment
    project_name = "${var.project_name}-${each.key}"
  })

  # Tags spécifiques
  custom_tags = {
    Name = "${var.project_name}-${each.key}"
    Role = each.value.role
  }
}

```

### 2. Adaptation des Outputs (`outputs.tf`)

Les outputs changent car on tape maintenant dans un module, plus dans une ressource directe.

```hcl
output "server_ips" {
  value = {
    for k, v in module.ec2_fleet : k => v.public_ip
  }
}

```

---

## Partie 5 : Déploiement GitOps

1. **Commit & Push** dans `lab01-audit`.
2. Observez le Pipeline.

**Ce qui va se passer lors du `terraform init` (dans les logs) :**

> $ terraform init -reconfigure
> Initializing the backend...
> Successfully configured the backend "http"! Terraform will automatically
> use this backend unless the backend configuration changes.
> Initializing modules...
> Downloading git::http://gitlab.wizetraining.com/f_mawaki/tf-module-aws-standard-ec2.git?ref=v1.0.0 for ec2_fleet...
> - ec2_fleet in .terraform/modules/ec2_fleet
> Initializing provider plugins...

Terraform va cloner le dépôt du module, se placer sur le tag `v1.0.0`, et utiliser ce code pour générer le plan.

**Le Plan :**

Si vous avez bien travaillé, le plan devrait indiquer : `No changes` (ou des changements mineurs de tags `CreatedBy`).
Pourquoi ? Parce que la logique infra est la même, on a juste déplacé le code ("Refactoring"). Terraform reconnaît que l'instance existante correspond à la définition du module.

Si vous rencontrez des problème d'authorisation ou d'accès au Module lors de la phase d'initialisation, vous verrez une erreur similaire. Vérifiez que le repo Git du module `tf-module-aws-standard-ec2` est bien public.

```
(...)
Initializing the backend...
Successfully configured the backend "http"! Terraform will automatically
use this backend unless the backend configuration changes.
Initializing modules...
Downloading git::http://gitlab.wizetraining.com/f_mawaki/tf-module-aws-standard-ec2.git?ref=v1.0.0 for ec2_fleet...
╷
│ Error: Failed to download module
│ 
│   on main.tf line 61:
│   61: module "ec2_fleet" {
│ 
│ Could not download module "ec2_fleet" (main.tf:61) source code from
│ "git::http://gitlab.wizetraining.com/f_mawaki/tf-module-aws-standard-ec2.git?ref=v1.0.0":
│ error downloading
│ 'http://gitlab.wizetraining.com/f_mawaki/tf-module-aws-standard-ec2.git?ref=v1.0.0':
│ /usr/bin/git exited with 128: Cloning into
│ '.terraform/modules/ec2_fleet'...
│ fatal: could not read Username for 'http://gitlab.wizetraining.com': No
│ such device or address
│ 
╵
Cleaning up project directory and file based variables 00:00
```

---

### 💡 Le Point Expert

**Pourquoi le Tag `?ref=v1.0.0` est critique pour une banque ?**

Imaginez que vous modifiez le module demain pour ajouter une règle "Instance Stop at Night".

* Sans le tag (`ref=main`), **tous** les projets de la banque subiraient ce changement dès leur prochain pipeline. Risque de régression massive.
* Avec le tag, le projet `lab01-audit` reste en `v1.0.0`. Il ne subira le changement que le jour où le développeur décidera explicitement de passer en `ref=v1.1.0`.

C'est ce qu'on appelle **l'Immutabilité des dépendances**.


## Partie 6 : Mettez à jour le module pour prendre en charge la clé SSH

Vous remarquerez que dans la version `v1.0.0` la clée SSH n'est pas prise en compte. Développez la version `v1.1.0` du module pour apporter cette fonctionnalité

---

## Solution finale

Vous pouvez télécharger le code de la solution prêt à être déployé ici : 

```bash
git clone http://gitlab.mpakoupete.com/root/terraform-correction-lab06.git
```

