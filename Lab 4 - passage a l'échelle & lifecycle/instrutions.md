# LAB 04 - Métadonnées de Production & Haute Disponibilité

**Contexte de l'Entreprise TERRA** : L'équipe Audit s'agrandit. Une seule machine ne suffit plus, ils ont besoin de trois serveurs aux profils différents (Front, API, Batch).
De plus, l'incident de la semaine dernière (suppression accidentelle d'une base de données) a traumatisé le management. Vous devez :

1. Déployer plusieurs instances dynamiquement sans dupliquer le code.
2. Garantir qu'une mise à jour de l'instance ne coupe pas le service (**Zero Downtime**).
3. Empêcher formellement Terraform de supprimer les données critiques.

**Objectifs** :

* Maîtriser les méta-arguments : `for_each`, `lifecycle`.
* Comprendre la stratégie de remplacement `create_before_destroy`.
* Protéger les ressources stateful avec `prevent_destroy`.

**Environnement** : GitLab CI (Toujours en mode GitOps).

---

## Partie 1 : Passage à l'échelle (for_each)

Plutôt que de copier-coller le bloc `resource` 3 fois (ce qui est une erreur de débutant), nous allons utiliser une boucle.

*Pourquoi `for_each` et pas `count` ?*

> Avec `count`, si vous supprimez l'instance n°1, l'instance n°2 devient la n°1 et Terraform la redémarre inutilement. Avec `for_each`, chaque instance a une clé unique (son nom), ce qui est beaucoup plus stable.

### 1. Définition de la flotte (`variables.tf`)

Ajouter une variable `servers` dans `variables.tf` pour inclure une "Map" définissant nos serveurs (`role` et `type` d'instance pour chaque).
Voir les types de variable dans [la documentation](https://developer.hashicorp.com/terraform/language/expressions/types#types)

<details>
<summary>Voir Correction</summary>

```hcl
# variables.tf

variable "servers" {
  description = "Map des serveurs à déployer avec leur dimensionnement"
  type        = map(object({
    type = string
    role = string
  }))
  default = {
    "front" = { type = "t3.micro", role = "frontend" }
    "api"   = { type = "t3.micro", role = "backend" }
    "batch" = { type = "t3.nano",  role = "worker" }
  }
}

```

</details>

### 2. Modification de la ressource (`main.tf`)

Transformez votre ressource unique en usine à instances.
* Incluer `For_each`
* Modifiez `instance_type` pour accéder aux propriétés de la variable Map `servers` définie
* Modifiez la variable `projet_name` du `User Data` pour afficher une valeur par instance (ex: `terra-audit-front`)

<details>
<summary>Voir Correction</summary>

```hcl
# main.tf

resource "aws_instance" "audit_vm" {
  # La boucle magique
  for_each = var.servers

  ami                    = var.ami_id

  # On accède aux propriétés de chaque objet via each.value
  instance_type          = each.value.type
  vpc_security_group_ids = [aws_security_group.audit_sg.id]

  # User data : On personnalise le script pour chaque serveur !
  user_data = templatefile("${path.module}/scripts/install_nginx.tftpl", {
    env_name     = var.environment
    project_name = "${var.project_name}-${each.key}" # ex: terra-audit-front
  })

}

```

</details>

### 3. Mise à jour des Outputs (`outputs.tf`)

Puisque `aws_instance.audit_vm` est maintenant une liste (collection), l'output précédent va planter. Il faut utiliser une boucle `for` pour tout afficher.

```hcl
output "server_ips" {
  description = "IPs publiques de tous les serveurs"
  # On crée une Map : Nom du serveur => IP
  value = {
    for server_name, server_obj in aws_instance.audit_vm : 
    server_name => server_obj.public_ip
  }
}

```

### 4. Push vers GitLab

```bash
# Vérifiez vos modifications
git status

# Ajoutez les nouveaux fichiers (notamment le dossier scripts/)
git add .

# Commitez
git commit -m "Lab 04 : Passage à l'échelle (for_each)"

# Envoyez à l'usine
git push

```

### 5. ajustement des noms

Vous constaterez dans la Console AWS que les VMs ont toutes le même nom.

Modifiez le `main.tf` pour que les noms (tags) soient dynamiques. Utilisez [la fonction `merge`](https://developer.hashicorp.com/terraform/language/functions/merge) pour fusionner les tags de `local` et les nom/tag personnalisés

<details>
<summary>Voir Correction</summary>

```hcl
  # Gestion des Tags dynamiques
  tags = merge(local.common_tags, {
    # On écrase le nom générique pour mettre un nom spécifique
    Name = "${var.project_name}-${each.key}" # ex: terra-audit-front
    Role = each.value.role
  })
```

</details>

### 6. Push vers GitLab

```bash
# Vérifiez vos modifications
git status

# Ajoutez les nouveaux fichiers (notamment le dossier scripts/)
git add .

# Commitez
git commit -m "Lab 04 : Passage à l'échelle (for_each) - tags à jours"

# Envoyez à l'usine
git push

```

---

## Partie 2 : Zéro Interruption (Lifecycle)

Par défaut, si vous modifiez le User Data (ex: changer le titre du site), Terraform **détruit** l'instance puis en **crée** une nouvelle.
**Conséquence** : 5 minutes de coupure de service.

Nous voulons l'inverse : Créer la nouvelle, attendre qu'elle soit prête, basculer le trafic, puis détruire l'ancienne.

### 1. Ajout du bloc Lifecycle

Ajoutez ce bloc à l'intérieur de votre ressource `aws_instance` dans `main.tf` :

```hcl
resource "aws_instance" "audit_vm" {
  # ... configuration existante ...

  lifecycle {
    # Règle d'or en Prod pour les serveurs Web sans état (Stateless)
    create_before_destroy = true
  }
}

```

**Attention** : Pour que cela fonctionne parfaitement, le nom de la ressource (si vous le forcez) doit être unique. Terraform ajoute souvent un suffixe aléatoire automatiquement lors de cette manœuvre, mais soyez vigilants sur les contraintes d'unicité (ex: noms de buckets S3).

---

## Partie 3 : La Ceinture de Sécurité (Prevent Destroy)

Imaginez que ces serveurs stockent des logs critiques sur un disque additionnel (EBS). Une erreur de manipulation dans le code (`terraform destroy` accidentel) serait catastrophique.

### 1. Création d'une ressource critique

Ajoutez ceci à `main.tf` :

```hcl
resource "aws_ebs_volume" "critical_logs" {
  availability_zone = "${var.aws_region}a"
  size              = 1 # 1 Go suffira pour le test

  tags = {
    Name = "CRITICAL-DATA-DO-NOT-DELETE"
  }

  lifecycle {
    # L'assurance vie de votre infra
    prevent_destroy = true
  }
}

```

---

## Partie 4 : Cycle de Vie GitOps (Validation)

C'est l'heure de vérité. Voyons comment GitLab gère ces changements structurels majeurs.

### 1. Push et Plan

```bash
git add .
git commit -m "Lab 04 : Scaling et Lifecycle rules"
git push

```

Allez voir le job **Plan** dans GitLab.

* Vous devriez voir `Plan: 3 to add, 1 to destroy`.
* *Pourquoi 1 destroy ?* Parce que vous avez renommé la ressource de "audit_vm" (scalaire) à "audit_vm[front]" (liste). Terraform doit remplacer l'ancienne par les nouvelles.

### 2. Test du "Prevent Destroy" (Simulation d'accident)

Une fois le pipeline passé (Apply vert), nous allons tenter de supprimer le volume critique via une modification de code.

1. **Localement**, commentez ou supprimez tout le bloc `resource "aws_ebs_volume" ...` dans `main.tf`.
2. Commitez et Pushez : `git commit -m "Oups suppression disque" && git push`.
3. Regardez le pipeline **échouer** au stade du **Plan**.

**Message attendu dans les logs du Job :**

> `Error: Instance cannot be destroyed`
> `Resource aws_ebs_volume.critical_logs has lifecycle.prevent_destroy set, but the plan calls for this resource to be destroyed.`

C'est la preuve que votre filet de sécurité fonctionne. Terraform refuse d'exécuter l'ordre de destruction.

---

### 💡 Questions pour vérifier la compréhension

**Q1 : Quelle est la différence fondamentale entre `count` et `for_each` dans le fichier `.tfstate` ?**

<details>
<summary>Réponse</summary>

* `count` identifie les ressources par un **index numérique** (0, 1, 2). Si on supprime la ressource 0, la 1 devient 0, ce qui force sa modification.
* `for_each` identifie les ressources par une **clé chaîne de caractères** ("front", "api"). L'ordre n'a pas d'importance, ce qui est beaucoup plus stable pour la prod.

</details>

**Q2 : Si j'utilise `create_before_destroy`, que se passe-t-il si mon quota d'IP Elastic ou de vCPU AWS est atteint ?**

<details>
<summary>Réponse</summary>

Le déploiement échouera. Comme Terraform essaie de créer les nouvelles ressources *avant* de supprimer les anciennes, vous consommez temporairement **le double** de ressources. Il faut prévoir ce dépassement de capacité (Surbooking).

</details>

**Q3 : Comment faire pour supprimer *vraiment* le volume EBS bloqué par `prevent_destroy` si on le veut vraiment ?**

<details>
<summary>Réponse</summary>

Il faut le faire en deux temps :

1. Modifier le code pour enlever `prevent_destroy = true` (ou le passer à `false`).
2. Appliquer ce changement (Apply).
3. Supprimer le code de la ressource.
4. Appliquer la destruction.
Cela évite les suppressions "par accident" en un seul commit.

</details>

---

## Solution finale

Vous pouvez télécharger le code de la solution prêt à être déployé ici : 

```bash
git clone https://github.com/wizetraining/terraform-correction.git
```

