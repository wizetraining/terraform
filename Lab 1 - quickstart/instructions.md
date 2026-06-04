# LAB 01 - Le "Quickstart" Standardisé (POC Audit)

Cet atelier pose les fondations de tout projet Terraform. Vous allez répondre à un besoin urgent (une machine d'audit) en remplaçant les actions manuelles (ClickOps) par du code structuré et maintenable.

**Contexte** : Un auditeur externe a besoin d'une machine temporaire pour analyser des logs. Il faut aller vite, mais l'équipe Sécurité interdit de coder des valeurs en dur (Hardcoding) et exige un nommage explicite des ressources.

**Pré-requis** :

* AWS CLI configuré avec des droits d'admin (ou PowerUser).
* Terraform installé sur votre poste (Local).
* Un éditeur de code (VSCode recommandé).

**Objectifs**:

* Structurer un projet Terraform selon les bonnes pratiques (séparation des fichiers).
* Bannir les "Magic Strings" en utilisant des variables typées.
* Sécuriser le réseau (Security Group) avec des règles strictes.
* Récupérer les informations de connexion (IP) via les Outputs.

---

## 📚 Documentation Officielle
Avant de commencer, voici les références indispensables à garder sous la main :
* [Documentation Terraform (Provider AWS)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
* [Guide des commandes CLI Terraform](https://developer.hashicorp.com/terraform/cli/commands)
* [Syntaxe HCL (HashiCorp Configuration Language)](https://developer.hashicorp.com/terraform/language)

---

## Partie 0 : Le "Crash Test" (Ce qu'il ne faut pas faire)

Avant de construire propre, nous allons lancer un déploiement "sale" pour identifier les risques immédiats d'une approche naïve.

### 1. Configuration de l'environnement

Pour que Terraform puisse piloter votre compte AWS, il a besoin de vos identifiants. Plutôt que de les stocker en dur (risque de fuite), nous les injectons dans la session courante.

Copiez-collez ces lignes dans votre terminal (remplacez par vos vraies clés) :

```bash
export AWS_ACCESS_KEY_ID="AKIAxxxxxxxxxxxxxxxx"
export AWS_SECRET_ACCESS_KEY="wJalxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export AWS_DEFAULT_REGION="us-east-1"

```

### 2. Le code "Anti-Pattern"

Créez un dossier `lab0` et mettez le fichier `main.tf` minimaliste avec des valeurs "en dur" (Hardcoded). mettez à jour le Tag Name en y mettant votre prenom (ex: `"dupond"`)

```hcl
# main.tf (Version naïve)
provider "aws" {
   region = "us-east-1"
}

resource "aws_instance" "example" {
   ami           = "ami-0230bd60aa48260c6"  # Amazon Linux 2 (us-east-1)
   instance_type = "t2.nano"

   tags = {
      Name = "<votre prenom>"
   }
}

```

### 3. Exécution et accès à AWS console pour vérification

Lancez le cycle de vie Terraform :

```bash
terraform init
terraform fmt
terraform plan
terraform apply -auto-approve

```

Accéder à la Console AWS pour constater l'instance que vous avez déployé dans la région `us-east-1`


### 4. Analyse Critique (Questions / Réponses)

Observez les logs de votre terminal et répondez aux questions suivantes pour comprendre pourquoi ce code ne passerait pas la validation de l'entreprise TERRA.

**Q1 : Quelle version du provider AWS a été téléchargée lors du `init` ? Quel est le risque ?**

<details>
<summary>Voir la réponse</summary>

Terraform a téléchargé la **dernière version disponible** (ex: `v6.32.1`).
**Le Risque :** Si demain HashiCorp sort la `v7.0` avec des changements de rupture (Breaking Changes), votre code risque de ne plus fonctionner. En production, on doit **toujours** fixer la version requise dans un bloc `required_providers` (voir Partie 1).

</details>

**Q2 : Si vous vouliez changer la région pour "eu-west-3" (Paris), que se passerait-il avec l'AMI indiquée ?**

<details>
<summary>Voir la réponse</summary>

Cela **planterait**. Les IDs d'AMI (ex: `ami-0230...`) sont spécifiques à une région. Une AMI de Virginie n'existe pas à Paris. C'est pourquoi coder des IDs en dur est une mauvaise pratique. Il faut utiliser des Variables ou des Data Sources.

</details>

**Q3 : Où est stocké l'état de votre infrastructure (le fichier qui sait que l'instance `i-0123...` correspond à votre code) ?**

<details>
<summary>Voir la réponse</summary>

Il est stocké dans un fichier **local** nommé `terraform.tfstate` apparu dans votre dossier.
**Le Risque :** Si vous perdez votre ordinateur, on perd la trace de l'infrastructure. Si un collègue veut modifier l'infra, il ne peut pas. C'est pourquoi on utilisera un **Backend distant** (S3/GitLab) plus tard.

</details>

---

*Une fois le test terminé, supprimez tout pour repartir propre : `terraform destroy -auto-approve`.*

---

## Partie 1 : Initialisation et Provider

Dans un contexte bancaire, on ne laisse jamais le hasard décider de la version des outils. Nous allons fixer les versions.

1. Créez un dossier nommé `lab01-audit` et placez-vous dedans.
2. Créez un fichier `providers.tf`.
3. Définissez le bloc `terraform` pour requérir le provider `hashicorp/aws` (version `~> 6.0`).
4. Configurez le bloc `provider "aws"` pour utiliser une région via une variable (ne mettez pas la région en dur ici).

<details>
<summary>Correction providers.tf</summary>

```hcl
# providers.tf ==> C'est ici que j'indique à terraform : Voici avec quel fournisseur de cloud je veux travailler et quelle version de ses outils je souhaite utiliser.

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
  required_version = ">= 1.14.0"
}

provider "aws" {
  region = var.aws_region
}

```

</details>

---

## Partie 2 : Définition des Variables (No Magic Strings)

L'utilisation de valeurs en dur est une dette technique immédiate. Nous allons tout externaliser.

1. Créez un fichier `variables.tf`.
2. Définissez les variables suivantes avec des descriptions claires et des types (Voir [la documentation sur les types](https://developer.hashicorp.com/terraform/language/block/variable#type)) :
* `aws_region` : Région de déploiement (défaut : `eu-west-3`).
* `project_name` : Nom du projet (ex: `terra-audit`; pour une meilleur visibilité, personaliser les noms pour y mettre votre prenom ex: `<votre prenom>-terra-audit`).
* `instance_type` : Type d'instance (ex: `t3.micro`).
* `auditor_ip` : L'IP autorisée à se connecter en SSH (utilisez votre IP publique ou `0.0.0.0/0` pour ce test seulement).
* `ami_id` : L'id de l'AMI à utiliser.



<details>
<summary>Correction variables.tf</summary>

```hcl
# variables.tf

variable "aws_region" {
  description = "La région AWS cible pour le déploiement"
  type        = string
  default     = "eu-west-3"
}

variable "project_name" {
  description = "Préfixe utilisé pour le nommage des ressources"
  type        = string
  default     = "<votre prenom>-terra-audit"
}

variable "instance_type" {
  description = "La taille de l'instance EC2"
  type        = string
  default     = "t3.micro"
}

variable "auditor_ip" {
  description = "CIDR autorisé pour la connexion SSH (IP de l'auditeur)"
  type        = string
  default     = "0.0.0.0/0" # À restreindre en prod !
}

# Pour simplifier ce Lab 1 sans Data Sources, on met l'AMI en variable
variable "ami_id" {
  description = "ID de l'AMI Ubuntu 22.04 (eu-west-3)"
  type        = string
  default     = "ami-05b5a865c3579bbc4" # Vérifiez que cette AMI existe dans votre région
}

```

</details>

---

## Partie 3 : Infrastructure et Sécurité

Maintenant que nous avons défini nos variables (dans la Partie 2), nous allons déployer les ressources en respectant le principe de **moindre privilège** et de **traçabilité**.
Nous allons déployer les ressources. Notez l'importance du nommage explicite dans le code (`audit_sg`, `audit_vm`) pour que n'importe quel développeur comprenne le but de la ressource sans lire les tags.

### 1. Le Security Group (Pare-feu) et L'Instance EC2

1. Créez un fichier `main.tf`.
2. **Sécurité d'abord** : Créez une ressource `aws_security_group` nommée `audit_sg` (Voir la [Documentation et des les exemples](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group#example-usage)).
* Autorisez le port `22` (Ingress) uniquement depuis `var.auditor_ip`.
* Autorisez tout le trafic sortant (Egress).
* Utilisez `var.project_name` dans le tag `Name`.
* Dans votre fichier `main.tf`, ne mettez aucune IP en dur pour l'`Ingress` ni de nom en dur.


3. **Compute** : Créez une ressource `aws_instance` nommée `audit_vm`.
* Utilisez l'AMI et le type d'instance définis dans les variables.
* Liez **dynamiquement** l'Instance EC2 à son Security Group => Attachez le Security Group créé juste avant (référencez-le par son attribut `.id`, pas son nom en dur !).
  * On peut se référer à la Section [Référence des Atributs de la Resources que vous souhaitez](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group#attribute-reference)
* Ajoutez des tags explicites : `Name`, `Owner`, `Environment`.


<details>
<summary>Correction main.tf</summary>

```hcl
# main.tf

# 1. Le Security Group (Pare-feu)
resource "aws_security_group" "audit_sg" {
  name        = "${var.project_name}-sg"
  description = "Allow SSH for Auditor"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.auditor_ip]      # Utilisation de la variable !
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"                  # -1 signifie tous les protocoles
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-sg"
  }
}

# 2. L'instance EC2
resource "aws_instance" "audit_vm" {
  ami             = var.ami_id
  instance_type   = var.instance_type
  
  # Liaison explicite avec le SG créé plus haut
  vpc_security_group_ids = [aws_security_group.audit_sg.id]

  tags = {
    Name        = "${var.project_name}-vm"
    Environment = "Audit"
    Owner       = "SecuTeam"
  }
}

```

</details>

---

### 2. Validation des Acquis (Questions / Réponses)

**Q1 : Dans la ressource `aws_instance`, pourquoi écrit-on `aws_security_group.audit_sg.id` et pas juste le nom "terra-audit-sg" entre guillemets ?**

<details>
<summary>Voir la réponse</summary>

C'est une **Référence**. Cela crée une **dépendance implicite**.

1. Terraform comprend qu'il doit créer le Security Group **AVANT** l'instance.
2. Si le Security Group change d'ID (recréation), Terraform mettra à jour l'instance automatiquement sans qu'on ait besoin de modifier le code.

</details>

**Q2 : Pourquoi avoir mis `0.0.0.0/0` en Egress (Sortie) est acceptable, alors qu'on l'interdit en Ingress (Entrée) ?**

<details>
<summary>Voir la réponse</summary>

En sécurité, on bloque tout ce qui rentre (pour éviter les attaques), mais on autorise souvent ce qui sort pour permettre au serveur de télécharger ses mises à jour (yum/apt) ou d'envoyer ses logs.

</details>

**Q3 : Si je change la variable `project_name` dans `variables.tf` et que je relance un `apply`, que va-t-il se passer pour le Security Group ?**

<details>
<summary>Voir la réponse</summary>

Terraform va devoir le **détruire et le recréer**.
Pourquoi ? Parce que le nom d'un Security Group AWS est immuable. Terraform détectera le changement ("Force replacement") et vous avertira dans le plan.

[La documentation ici](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group#name-1) révèle une notion importe sur le champ `name` :  **(Optional, Forces new resource)**

</details>


## Partie 4 : Restitution (Outputs)

L'auditeur ne doit pas avoir besoin de se connecter à la console AWS pour trouver son IP. Terraform doit lui fournir cette information à la fin du déploiement.

Pour connaitres les champs qu'on peut afficher, on peut se référer à la Section [Référence des Atributs de la Resources que vous souhaitez](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance#attribute-reference)

1. Créez un fichier `outputs.tf`.
2. Exposez l'ID de l'instance créée.
3. Exposez l'IP Publique de l'instance créée.

<details>
<summary>Correction outputs.tf</summary>

```hcl
# outputs.tf

output "audit_vm_id" {
  description = "ID de l'instance générée"
  value       = aws_instance.audit_vm.id
}

output "audit_vm_public_ip" {
  description = "IP Publique pour la connexion SSH"
  value       = aws_instance.audit_vm.public_ip
}

```

</details>

---

## Partie 5 : Cycle de vie (Workflow)

C'est le moment de tester votre code.

1. **Initialisation** : Téléchargez le provider AWS.
2. **Formatage** : Assurez-vous que votre code est propre (canonical).
3. **Planification** : Vérifiez ce que Terraform compte faire. *Combien de ressources seront ajoutées ?*
4. **Application** : Lancez le déploiement. Tapez `yes` pour confirmer.
5. **Vérification** : Récupérez l'IP dans les outputs et tentez de pinguer (ou juste vérifier la console).

<details>
<summary>Correction Commandes</summary>

```bash
# 1. Initialiser le backend et les providers
terraform init

# 2. Reformater le code proprement. One perd pas le temps à supprimer les espcaes par exemple ou arranger le code soit même. Cette commande le fait automatiquement pour nous.
terraform fmt

# 3. Valider la syntaxe
terraform validate

# 4. Voir le plan d'exécution (Dry Run)
terraform plan

# 5. Appliquer
terraform apply
# (Tapez 'yes' à la demande)

# Résultat attendu :
# Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
#
# Outputs:
# audit_vm_id = "i-0123456789abcdef0"
# audit_vm_public_ip = "13.37.xxx.xxx"

```

</details>

---

## Solution finale

Vous pouvez télécharger le code de la solution prêt à être déployé ici : 

```bash
git clone http://gitlab.mpakoupete.com/root/terraform-correction-lab01.git
```