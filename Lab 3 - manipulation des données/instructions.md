# LAB 03 - Data Engineering & User Data

**Contexte de l'Entreprise TERRA** : L'instance "Audit" déployée au Lab 02 est fonctionnelle mais... vide. L'équipe Application a besoin qu'un serveur Web (Nginx) soit installé automatiquement au démarrage. De plus, l'équipe Architecture (CMDB) rejette vos tags actuels : ils exigent un format normalisé strict (Trigramme Majuscule) pour le `Environment` (ex: "production" doit devenir "PRD").

**Objectifs** :

1. **Automatiser l'installation logicielle** sans SSH (User Data).
2. **Transformer la donnée** utilisations des [fonctions Terrafom (Voir Documentation)](https://developer.hashicorp.com/terraform/language/functions) : (Strings) pour respecter les normes de nommage.
3. **Exporter la configuration** sous format machine (JSON).

**Environnement** : GitLab CI (Modification locale -> Push -> Pipeline).

---

## Partie 1 : Injection de Script (Filesystem)

Pour installer Nginx au démarrage, nous allons utiliser la fonctionnalité `user_data` d'AWS (Cloud-Init). Plutôt que d'écrire le script BASH en dur dans le Terraform (ce qui est sale), nous allons le mettre dans un fichier à part et l'injecter dynamiquement.

### 1. Création du Template de Script

À la racine de votre projet `lab01-audit` (qui est devenu votre repo git), créez un dossier `scripts/`.
Dans ce dossier, créez un fichier `install_nginx.tftpl` (l'extension `.tftpl` est une convention pour "Terraform Template").

```bash
#!/bin/bash
# scripts/install_nginx.tftpl

# Mise à jour et installation
sudo apt-get update -y
sudo apt-get install -y nginx

# Création d'une page d'accueil personnalisée
# Notez la syntaxe avec caractère spéciaux: C'est Terraform qui va remplacer ça AVANT d'envoyer le script à AWS
echo "<h1>Site Audit - Deploye par Terraform</h1>" > /var/www/html/index.html
echo "<p>Environment : ${env_name}</p>" >> /var/www/html/index.html
echo "<p>Project : ${project_name}</p>" >> /var/www/html/index.html

# Démarrage du service
sudo systemctl start nginx
sudo systemctl enable nginx

```

### 2. Modification de `main.tf` (Fonction `templatefile`)

Nous devons dire à Terraform : "Lis ce fichier, remplace les variables `${...}` dedans, et injecte le résultat dans l'instance".

Modifiez la ressource `aws_instance` dans `main.tf` pour ces fins:

**Astuces :** Utiliser la [fonction built-in `templatefile`](https://developer.hashicorp.com/terraform/language/functions/templatefile)


<details>
<summary>Voir Correction</summary>

```hcl
(...)
  # --- AJOUT LAB 03 : User Data ---
  # La fonction templatefile prend 2 arguments :
  # 1. Le chemin du fichier
  # 2. Un objet map {} contenant les variables à remplacer

  user_data = templatefile("${path.module}/scripts/install_nginx.tftpl", {
    env_name     = var.environment    # Nous allons créer cette variable juste après
    project_name = var.project_name
  })
  
  # Pour que le User Data s'exécute à chaque changement (optionnel mais utile en Lab)
  user_data_replace_on_change = true
  # --------------------------------
(...)

```

</details>


### 3. Ajout de la variable manquante

Dans `variables.tf`, ajoutez la variable d'environnement (si elle n'existe pas déjà) :

<details>
<summary>Voir Correction</summary>

```hcl
variable "environment" {
  description = "L'environnement de déploiement (ex: production, staging, audit)"
  type        = string
  default     = "audit" 
}

```

</details>


### 4. Ouverture du Flux HTTP

Pour voir le résultat, il faut ouvrir le port 80. Modifiez votre `aws_security_group` dans `main.tf` pour ajouter une règle `ingress` :

```hcl
  # ... règle SSH existante ...

  # Ajout HTTP
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Web ouvert à tous
  }

```

### 5. Push vers GitLab

```bash
# Vérifiez vos modifications
git status

# Ajoutez les nouveaux fichiers (notamment le dossier scripts/)
git add .

# Commitez
git commit -m "Lab 03 : Nginx UserData"

# Envoyez à l'usine
git push

```

### 6. modification :

Modifier le fichier `install_nginx.tftpl` et ou les variables d'environnement et observez le changement.

---

## Partie 2 : Normalisation des Données (String Functions)

La CMDB est stricte : Le tag `Environment` doit être un **Trigramme** (3 lettres) en **Majuscules**.

* Entrée utilisateur : "audit" -> Sortie Tag : "AUD"
* Entrée utilisateur : "production" -> Sortie Tag : "PRO"

Nous allons utiliser le bloc de type `locals` et les fonctions de manipulation de chaînes.
Voir la documentation : [Locals Terraform](https://developer.hashicorp.com/terraform/language/values/locals)

**Astuces :** 
* Utiliser la [Function `upper`](https://developer.hashicorp.com/terraform/language/functions/upper)
* Utiliser la [Function `substr`](https://developer.hashicorp.com/terraform/language/functions/substr)

### 1. Définition de la logique (`main.tf`)

Ajoutez un bloc `locals` au début de votre `main.tf`. Les `locals` sont comme des variables temporaires internes, calculées à la volée.

```hcl
locals {
  # 1. On met tout en majuscule : "audit" -> "AUDIT"
  env_upper = upper(var.environment)

  # 2. On coupe pour ne garder que les 3 premiers caractères : "AUDIT" -> "AUD"
  env_trigram = substr(local.env_upper, 0, 3)

  # 3. On formate un nom standardisé complet
  common_tags = {
    Name        = "${var.project_name}-vm"
    Owner       = "TER-Secu"
    Environment = local.env_trigram # On utilise la version transformée
    ManagedBy   = "Terraform"
  }
}

```

### 2. Application des Tags

Modifiez la ressource `aws_instance` pour utiliser ces tags calculés :

<details>
<summary>Voir Correction</summary>

```hcl
resource "aws_instance" "audit_vm" {
  # ... le reste de la config ...

  # Au lieu d'écrire les tags en dur, on injecte la map calculée
  tags = local.common_tags
}

```

</details>


### 3. Push vers GitLab

```bash
# Vérifiez vos modifications
git status

# Ajoutez les nouveaux fichiers (notamment le dossier scripts/)
git add .

# Commitez
git commit -m "Lab 03 : Tags normalisés"

# Envoyez à l'usine
git push

```

---

## Partie 3 : Encodage et Output (Encoding Functions)

L'auditeur souhaite recevoir un fichier JSON récapitulatif de la configuration déployée pour ses propres outils d'analyse automatisée.

### 1. Création de l'Output JSON (`outputs.tf`)

Nous allons utiliser `jsonencode()` pour transformer nos données Terraform en objet JSON valide.

Modifiez `outputs.tf` :

```hcl
# Les outputs existants...

output "audit_config_json" {
  description = "Configuration complète au format JSON pour la CMDB"
  value       = jsonencode({
    id             = aws_instance.audit_vm.id
    ip_address     = aws_instance.audit_vm.public_ip
    environment    = local.env_trigram  # On récupère la valeur calculée
    project        = var.project_name
    deployment_date = timestamp()       # Fonction temporelle bonus
  })
}

```

### 2. Push vers GitLab

```bash
# Vérifiez vos modifications
git status

# Ajoutez les nouveaux fichiers (notamment le dossier scripts/)
git add .

# Commitez
git commit -m "Lab 03 : Output JSON"

# Envoyez à l'usine
git push

```

### 3. Observation du Pipeline

1. Allez sur GitLab > **Build** > **Pipelines**.
2. Dans le job **Plan**, cliquez pour voir les logs. Cherchez les lignes suivantes pour valider votre logique :
* `tags`: Vous devriez voir `Environment = "AUD"` (et non "audit").
* `user_data`: Vous verrez une chaîne encodée (hash) indiquant qu'il y a un changement.

3. Si le plan est correct, lancez le job **Apply** (Manuel).

### 4. Vérification Finale (Recette)

Une fois le déploiement terminé (vert) :

1. Regardez les logs du job `apply` : Copiez la valeur de l'output `audit_config_json`. C'est du JSON valide !
2. Récupérez l'IP publique.
3. Ouvrez votre navigateur : `http://<IP_PUBLIQUE>`.
* Vous devriez voir : **"Site Audit - Deployé par Terraform"**
* Et en dessous : **"Environment : audit"** (Notez que dans la page HTML, c'est la variable `env_name` brute qu'on a passée, pas le trigramme, ce qui prouve qu'on maîtrise les deux flux !).

---

### 💡 Questions pour vérifier la compréhension

* **Q1** : Pourquoi utiliser `templatefile` plutôt que de copier le texte dans `main.tf` ?

<details><summary>Réponse</summary>Pour la lisibilité (séparation du code HCL et du code Bash) et pour pouvoir injecter des variables Terraform dynamiques dans le script.</details>

* **Q2** : Si je change la variable `environment` pour "development", quelle sera la valeur du Tag sur AWS ? (Testez le changement et observez le comportement)

<details><summary>Réponse</summary>`upper(&quot;development&quot;)` -> "DEVELOPMENT". `substr(..., 0, 3)` -> **"DEV"**.</details>

* **Q3** : Le script `user_data` s'exécute-t-il à chaque `terraform apply` ?

<details><summary>Réponse</summary>Non, par défaut uniquement à la **création** de l'instance. Si vous voulez qu'il se relance, il faut forcer la recréation de l'instance (ce que fait l'option `user_data_replace_on_change = true` que nous avons ajoutée).</details>


---

## Solution finale

Vous pouvez télécharger le code de la solution prêt à être déployé ici : 

```bash
git clone https://github.com/wizetraining/terraform-correction.git
```