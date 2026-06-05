# LAB 07 - Durcissement du State (S3 Native Locking)

**Contexte de l'Entreprise TERRA** :
L'audit de sécurité a révélé une vulnérabilité potentielle : votre fichier d'état (`terraform.tfstate`) est géré par GitLab. Bien que pratique, la politique de la banque exige que les données d'infrastructure critiques soient stockées dans des **Buckets S3 chiffrés (KMS)** et contrôlés par nos équipes IAM, avec un **verrouillage (Locking)** strict pour éviter les corruptions lors des déploiements concurrents.

⚠️ **Note importante :** Le vérrouillage de l'état par DynamoDB est déprécié et ne sera plus supporté dans les prochaines versions mineurs. Lire [la documentation ici](https://developer.hashicorp.com/terraform/language/backend/s3#state-locking). Pour activer le verrouillage d'état S3 (S3 Native Locking), utilisez l'argument facultatif `use_lockfile`.

**Objectifs** :

1. **Bootstrap** : Créer l'infrastructure de backend (Bucket + Table) via CLI.
2. **Migration** : Déplacer le state existant (GitLab HTTP) vers AWS S3 sans perte de données.
3. **State Locking** : Prouver que deux personnes ne peuvent pas modifier l'infra en même temps.

**Environnement** :

* **Action Admin** : Terminal Local (pour la migration).
* **Action User** : GitLab CI (pour l'utilisation quotidienne).

---

## Partie 1 : Bootstrap - creation du bucket S3 pour le backend

On ne peut pas utiliser Terraform pour créer le Bucket qui va stocker l'état de Terraform (paradoxe de l'œuf et la poule). Nous devons utiliser le script AWS CLI ("Bootstrap").

**Consigne (Terminal Local) :**

1. Assurez-vous d'avoir vos variables d'environnement AWS chargées (`export AWS_ACCESS_KEY_ID=...`).
2. Lancez ces commandes pour créer le backend :

```bash
# 1. Créer un nom unique (les buckets S3 sont uniques mondialement)
BUCKET_NAME="<votre prenom>-wizetraining-terra-tfstate-$(date +%Y-%m-%d)-$RANDOM"
echo "Votre Bucket sera : $BUCKET_NAME"

# 2. Créer le Bucket S3 (Stockage)
aws s3api create-bucket --bucket $BUCKET_NAME --region eu-west-3 \
    --create-bucket-configuration LocationConstraint=eu-west-3

# 3. Activer le Versioning (OBLIGATOIRE en Prod pour rollback le state)
aws s3api put-bucket-versioning --bucket $BUCKET_NAME --versioning-configuration Status=Enabled

```

---

## Partie 2 : La Migration Automatisée (Via GitLab CI)

C'est l'étape délicate. Nous devons dire à Terraform : *"Arrête de parler au Backend HTTP de GitLab, prends tes valises et va sur S3."*

Puisque nous sommes dans un contexte **GitOps**, nous n'allons pas faire cette migration depuis votre poste local (ce qui nécessiterait de configurer des tokens d'accès complexes), mais nous allons laisser le **Runner GitLab** faire le travail pour nous. Il possède déjà les accès à l'ancien state (HTTP) et au nouveau (S3 via vos variables AWS).

### 1. Configuration du nouveau Backend (`backend.tf`)

Dans votre dossier `lab01-audit` (en local) :

1. Créez un fichier dédié `backend.tf`.
2. Ajoutez-y la configuration S3.
3. **Important :** Supprimez tout bloc `backend "http"` qui pourrait traîner dans `main.tf` ou `providers.tf`.

```hcl
# backend.tf

terraform {
  backend "s3" {
    bucket         = "<votre prenom>-wizetraining-terra-tfstate-xxxxx"      # <--- REMPLACEZ PAR VOTRE NOM DE BUCKET (généré partie 1)
    key            = "audit-app/terraform.tfstate"
    region         = "eu-west-3"
    use_lockfile   = true   # Active le verrouillage natif S3 (Pas besoin de DynamoDB)
    encrypt        = true
  }
}

```

### 2. Création du Job de Migration (`.gitlab-ci.yml`)

Nous allons ajouter un job temporaire "Magique" qui va effectuer la bascule.

Ouvrez `.gitlab-ci.yml` et ajoutez le stage `migration` ainsi que le job `migrate_to_s3` suivant :

 **Important :** Activez le mode manuel de toutes les autres tâches `validate`, `plan`, `apply`, `destroy` car ces tâche hérite du `before_script` Global et donc annulerait notre effort de migration

```yaml
stages:
  - migration     # <--- Nouveau stage temporaire (à placer en premier)
  - validate
  - plan
  - apply
  - destroy

# ... (Le reste de votre fichier ne change pas) ...

# -----------------------------------------------------------------------------
# JOB DE MIGRATION UNIQUE (HTTP -> S3)
# -----------------------------------------------------------------------------
migrate_to_s3:
  stage: migration
  # On écrase le before_script global pour contrôler l'initialisation manuellement
  before_script:
    - terraform --version
  script:
    # Terraform détecte le changement de backend et on force la copie vers S3
    - terraform init -migrate-state -force-copy
    
  when: manual # Sécurité : Action manuelle unique
  allow_failure: true

```

### 3. Exécution de la Migration

1. **Commit & Push :** Envoyez ces modifications sur GitLab.
```bash
git add .
git commit -m "Setup migration job S3"
git push

```


2. **Lancement :**
* Allez dans GitLab > **Build** > **Pipelines**.
* Cliquez sur le pipeline en cours (qui devrait être "blocked" ou en attente).
* Localisez le job **`migrate_to_s3`** et cliquez sur le bouton **Play** (▶️).


3. **Vérification :**
* Attendez que le job soit vert ✅.
* Ouvrez les logs du job. Vous devriez voir : 
> 
> ```bash
> $ terraform init -migrate-state -force-copy
> Initializing the backend...
> Terraform detected that the backend type changed from "http" to "s3".
> Successfully configured the backend "s3"! Terraform will automatically
> use this backend unless the backend configuration changes.
> ```
* **Preuve ultime :** Allez dans la console AWS S3, ouvrez votre bucket. Vous devez voir le fichier `audit-app/terraform.tfstate`. Téléchargez le fichier State

---

## Partie 3 : Nettoyage du Pipeline

La migration est terminée. Le fichier state est sur S3. Il faut maintenant nettoyer le pipeline pour qu'il n'utilise plus jamais l'ancien backend HTTP.

**Consigne :**

1. Ouvrez `.gitlab-ci.yml`.
2. **Supprimez** le job temporaire `migrate_to_s3` et le stage `migration`.
3. **Supprimez** (ou commentez) toutes les variables liées à l'ancien backend HTTP (`TF_HTTP_ADDRESS`, `TF_HTTP_PASSWORD`, etc.).


<details>
<summary>Correction .gitlab-ci.yml (nettoyé)</summary>

```yaml
# =============================================================================
# PIPELINE TERRAFORM - FINAL (BACKEND S3)
# =============================================================================
# Ce pipeline est nettoyé de toute trace de migration.
# Il utilise exclusivement le backend S3 configuré dans `backend.tf`.

# --- 1. CONFIGURATION DU RUNNER ---
default:
  tags:
    - linux # Sélectionne un runner Linux (requis pour Docker)

# --- 2. IMAGE DOCKER ---
image:
  name: hashicorp/terraform:latest
  entrypoint: [""] # Hack technique : Surcharge l'entrée pour que GitLab puisse lancer ses scripts

# --- 3. VARIABLES GLOBALES ---
variables:
  # Cible AWS : La région par défaut
  # NOTE : Les clés AWS_ACCESS_KEY_ID et SECRET ne sont PAS ici par sécurité.
  # Elles sont injectées masquées via "Settings > CI/CD > Variables".
  AWS_DEFAULT_REGION: "eu-west-3"
  AWS_REGION: "eu-west-3"

  # NOTE DE MIGRATION :
  # Toutes les variables TF_HTTP_* ont été supprimées.
  # Terraform est désormais forcé de lire la configuration S3 dans le fichier `backend.tf`.

# --- 4. CACHE (Optimisation) ---
cache:
  key: default
  paths:
    - .terraform # Évite de retélécharger les plugins AWS (environ 300Mo) à chaque job

# --- 5. INITIALISATION (Exécuté avant chaque job) ---
before_script:
  - terraform --version
  # On lance l'init simple.
  # Terraform va lire `backend.tf`, voir qu'il faut utiliser S3,
  # et se connecter au Bucket grâce aux variables CI/CD AWS.
  - terraform init -reconfigure

# --- 6. DÉFINITION DES ÉTAPES ---
stages:
  - validate
  - plan
  - apply
  - destroy

# =============================================================================
# JOBS DU PIPELINE
# =============================================================================

# Étape 1 : Vérification de la syntaxe
validate:
  stage: validate
  script:
    - terraform validate

# Étape 2 : Planification (Lecture seule)
plan:
  stage: plan
  script:
    # Génère le plan et le sauvegarde dans un fichier binaire "tfplan"
    # C'est ici que Terraform compare le State S3 avec votre Code.
    - terraform plan -out=tfplan
  artifacts:
    # L'artefact est vital : il transmet le fichier "tfplan" au job suivant.
    # Cela garantit que ce qu'on applique est EXACTEMENT ce qu'on a validé.
    paths:
      - tfplan

# Étape 3 : Application (Modification de l'infra)
apply:
  stage: apply
  script:
    # Applique le plan sans redemander confirmation (déjà validé par l'humain via le clic)
    - terraform apply -auto-approve tfplan
  when: manual # Sécurité : Production nécessitant une action humaine

# Étape 4 : Destruction (Nettoyage)
destroy:
  stage: destroy
  script:
    - terraform destroy -auto-approve
  when: manual # Sécurité absolue
```

</details>

5. **Commit & Push & Verify :**

```bash
git add .
git commit -m "Cleanup migration tools"
git push

```


6. **Validation Finale :** Regardez le job **`plan`** du nouveau pipeline.
* Il doit réussir ✅.
* Il doit indiquer **`No changes`**.
* *Cela prouve que Terraform a bien lu le state sur S3 et qu'il correspond exactement à votre infrastructure actuelle.*


---

## Partie 4 : Simulation de Conflit (State Locking)

C'est ici qu'on justifie l'utilisation de `use_lockfile = true` (S3 Native Locking). Nous allons prouver que S3 empêche deux ingénieurs de modifier l'infra en même temps.

**Le Scénario :**

* **Ingénieur A (Vous, localement)** : Lance un `apply` qui pose un verrou sur S3.
* **Ingénieur B (Le Pipeline GitLab)** : Tente de lancer un déploiement en même temps.

**Pré-requis Local :**
Puisque vous avez migré le state sur S3, votre dossier local `.terraform` est obsolète.
Dans votre terminal :

```bash
rm -rf .terraform/     # On supprime l'ancien cache
terraform init         # On se connecte au bucket S3 (Authentification AWS requise)

```

### Consignes :

1. **GitLab (Ingénieur B)** :

Modifiez par exemple le template user data `install_nginx.tftpl` puis poussez le code dans le repo Git pour que le pipeline s'enclenche. **Attendre que l'Ingenieur A soit pret avant de valider le `apply`**

```bash
git add .
git commit -m "template chnaged - test state locking"
git push

```

* Allez sur GitLab.
* Relancez manuellement le pipeline (bouton "Run pipeline" en haut à droite).
* Regardez les logs du job **Plan**.

2. **Terminal Local (Ingénieur A)** :

Modifiez à nouveau le code terraform (par exemple le template user data) et lancez une commande qui prend le verrou :

```bash
terraform apply

```

*Terraform affiche le plan et attend votre "yes". **NE RÉPONDEZ RIEN.** Laissez le curseur clignoter.*
*(À ce moment précis, un fichier `.tflock` caché a été créé dans votre bucket S3 (vous pouvez le consulter)).*

3. **Résultat Attendu dans GitLab :**

Le job doit échouer ❌ avec une erreur explicite :

```text
│ Error: Error acquiring the state lock
│ 
│ Error message: operation error S3: PutObject, https response error
│ StatusCode: 412, RequestID: B52YZGRFW69184XP, HostID:
│ idR6tK4jKAtbUUgFI39u7M5q9ISp/+qXFkVJivNgzKonsQy3Vw5l9hIJQ0maJcwkhDL9H5oA7nMTUFUqK1JGnQHsZFcMlJBq,
│ api error PreconditionFailed: At least one of the pre-conditions you
│ specified did not hold
│ Lock Info:
│   ID:        5009e756-b632-152e-d9d2-79325e4b0acf
│   Path:      testlab-wizetraining-terra-tfstate-2026-02-16-23019/audit-app/terraform.tfstate
│   Operation: OperationTypeApply
│   Who:       plb@localhost.localdomain
│   Version:   1.14.5
│   Created:   2026-02-16 22:59:16.449643041 +0000 UTC
│   Info:      
│ 
│ 
│ Terraform acquires a state lock to protect the state from being written
│ by multiple users at the same time. Please resolve the issue above and try
│ again. For most commands, you can disable locking with the "-lock=false"
│ flag, but this is not recommended.
```

### Conclusion :

Le pipeline a été bloqué par S3. L'intégrité de votre infrastructure est protégée.

3. **Libération :**

* Revenez sur votre terminal.
* Tapez `no` (ou Ctrl+C) pour annuler l'apply.
* Relancez le job dans GitLab : il passera maintenant sans erreur.

---

### 💡 Le Point Expert

**Q1 : Quelle est la différence entre DynamoDB Locking et S3 Native Locking ?**

<details>
<summary>Réponse</summary>
Pendant des années, S3 ne supportait pas le verrouillage atomique. On devait utiliser une table DynamoDB externe pour gérer les verrous. Depuis fin 2024 (Terraform 1.10+), S3 gère cela nativement via <code>use_lockfile = true</code>. C'est plus simple (un seul service) et moins cher, tout en étant aussi robuste.
</details>

**Q2 : Si le pipeline crashe pendant un apply, comment retirer le verrou S3 ?**

<details>
<summary>Réponse</summary>
Terraform fournit la commande <code>terraform force-unlock <LOCK_ID></code>. Avec le backend S3 natif, cela supprime simplement le fichier de lock caché dans le bucket. Attention : à n'utiliser que si vous êtes sûr que le processus bloquant est mort !
</details>

**Q3 : Pourquoi ne pas stocker le state dans le Git du projet ?**

<details>
<summary>Réponse</summary>
Pour deux raisons fatales :

1. <b>Sécurité :</b> Le tfstate contient tous les secrets en clair (mots de passe DB, clés API). Le mettre sur Git = Fuite de données.
2. <b>Locking :</b> Git ne gère pas le verrouillage. Deux commits simultanés = Conflit de merge sur un fichier JSON illisible = State corrompu = Infra cassée.
</details>