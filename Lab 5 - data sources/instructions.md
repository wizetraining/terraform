# LAB 05 - Data Sources & Golden Images

**Contexte de l'Entreprise TERRA** :
L'équipe de Sécurité (SecOps) a émis une non-conformité majeure sur votre projet.
**Le problème** : Vous utilisez un ID d'AMI codé en dur (`ami-05b5...`).
**Le risque** : Cette image vieillit, ne reçoit plus de patchs de sécurité et devient vulnérable. De plus, cet ID est spécifique à la région Paris ; si on déploie ailleurs, le code casse.

**La demande** : Terraform doit interroger dynamiquement AWS pour récupérer la dernière version de l'image "Golden" (Ubuntu 22.04) validée, à chaque déploiement.

**Objectifs** :

* Comprendre la différence entre `resource` (Créer) et `data` (Lire).
* Utiliser `data "aws_ami"` avec des filtres précis. (Voir [la documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami))
* Nettoyer le code en supprimant les variables d'AMI statiques.

**Environnement** :

* **Partie 1** : Local (Sandbox) pour tester la requête.
* **Partie 2** : GitLab CI (Mise à jour de la Prod).

---

## Partie 1 : Le "Sandbox" (Test de la requête)

Avant de toucher au code de production (qui pilote 3 serveurs !), nous devons mettre au point notre filtre de recherche. Si le filtre est mauvais, Terraform ne trouvera rien ou, pire, prendra une mauvaise image.

**Consigne :**

1. Créez un **nouveau dossier** temporaire sur votre bureau nommé `sandbox-data`.
2. Créez un fichier `main.tf` minimaliste à l'intérieur.
3. Ajoutez le provider AWS (région `eu-west-3`).
4. Écrivez un bloc `data "aws_ami"` pour trouver la dernière Ubuntu 22.04 fournie par Canonical (`099720109477`).
* Astuce filtre : Le nom ressemble à `ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*`.


5. Ajoutez un `output` pour afficher l'ID trouvé.

*Note : N'oubliez pas d'exporter vos clés AWS dans votre terminal (`export AWS_ACCESS_KEY_ID=...`) pour ce test local. si ce n'est pas encore fait*

<details>
<summary>Correction main.tf (Sandbox)</summary>

```hcl
provider "aws" {
  region = "eu-west-3"
}

# On interroge l'API AWS : "Donne-moi la liste des images..."
data "aws_ami" "ubuntu_latest" {
  most_recent = true
  owners      = ["099720109477"] # ID officiel de Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

output "ami_id_found" {
  value = data.aws_ami.ubuntu_latest.id
}

```

</details>

**Validation :**

Lancez `terraform init` puis `terraform apply`.
Terraform ne créera rien (0 resources added), mais affichera l'ID dans les Outputs.
Exemple : `ami_id_found = "ami-05b5a865c3579bbc4"` (ou une version plus récente).

---

## Partie 2 : Intégration en Production (GitOps)

Maintenant que la requête est validée, nous allons l'intégrer dans le projet principal `lab01-audit` (celui géré par GitLab).

### 1. Nettoyage des Variables (`variables.tf`)

Puisque l'AMI sera dynamique, la variable `ami_id` devient obsolète. C'est du "Dead Code".

* Ouvrez `variables.tf`.
* **Supprimez** (ou commentez) entièrement le bloc `variable "ami_id"`.

### 2. Ajout du Data Source (`main.tf`)

* Ouvrez `main.tf`.
* Ajoutez le bloc `data "aws_ami"` (celui de la partie 1) tout en haut du fichier, avant les ressources.

### 3. Mise à jour de la flotte d'instances (`main.tf`)

Il faut dire à nos instances d'utiliser le résultat de la recherche plutôt que la variable supprimée.

* Modifiez la ressource `aws_instance "audit_vm"`.
* Changez la ligne `ami = var.ami_id` par la référence au data source.

<details>
<summary>Correction main.tf (Mise à jour)</summary>

```hcl
# --- DATA SOURCE (Lecture) ---
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# --- RESOURCES (Création) ---
resource "aws_security_group" "audit_sg" {
  # ... (inchangé)
}

resource "aws_instance" "audit_vm" {
  for_each = var.servers

  # C'EST ICI QUE ÇA CHANGE :
  ami           = data.aws_ami.ubuntu.id # Référence dynamique
  instance_type = each.value.type
  
  # ... le reste (sg, lifecycle, user_data) reste inchangé
}

```

</details>

---

## Partie 3 : Cycle de Vie GitOps (Le Piège de l'Immutabilité)

C'est le moment critique. Nous allons pousser le code.

**Consigne :**

1. Faites le `git add .`, `git commit` et `git push`.
2. Allez observer le job **Plan** dans GitLab.

### Analyse du Plan (Question Expert)

Vous risquez de voir quelque chose comme :
`Plan: 3 to add, 3 to destroy.`

**Pourquoi Terraform veut-il détruire vos serveurs ?**

<details>
<summary>Voir la réponse</summary>

L'AMI que vous aviez codée en dur au Lab 01 date peut-être d'il y a quelques mois.
Le `data source` récupère l'image publiée **aujourd'hui** (la toute dernière version).

Pour AWS, changer l'AMI d'une instance est impossible sans la recréer.
Terraform détecte que `ami-vieux` != `ami-neuf`, donc il doit :

1. Créer les nouveaux serveurs (avec la nouvelle image).
2. Basculer le trafic (si on avait un Load Balancer).
3. Détruire les anciens serveurs.

Grâce au bloc `lifecycle { create_before_destroy = true }` ajouté au Lab 04, cette opération se fera sans interruption brutale !

</details>

### Action finale

Validez le job **Apply** (Manuel) dans GitLab pour effectuer la mise à jour de votre flotte vers la dernière version d'Ubuntu.

---

### 💡 Questions pour vérifier la compréhension

**Q1 : Quelle est la différence entre `data "aws_ami"` et `resource "aws_ami"` ?**

<details>
<summary>Réponse</summary>

* `resource "aws_ami"` : Sert à **créer** une nouvelle image (ex: après avoir configuré un serveur avec Packer), pour la stocker dans votre compte.
* `data "aws_ami"` : Sert uniquement à **chercher/lire** une image qui existe déjà (fournie par AWS ou Canonical).

</details>

**Q2 : Si je relance le pipeline dans 6 mois, que se passera-t-il ?**

<details>
<summary>Réponse</summary>

Si Canonical a sorti une nouvelle version d'Ubuntu entre temps, le `data source` récupérera le nouvel ID. Terraform détectera une différence et proposera de mettre à jour (reconstruire) vos serveurs. C'est ainsi qu'on gère le "Patch Management" automatisé avec Terraform !

</details>

---

## Solution finale

Vous pouvez télécharger le code de la solution prêt à être déployé ici : 

```bash
git clone https://github.com/wizetraining/terraform-correction.git
```