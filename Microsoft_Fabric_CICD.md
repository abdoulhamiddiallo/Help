Bonjour,

J’espère que vous allez bien également.

Voici une proposition d’implémentation que j’utilise généralement sur des projets Microsoft Fabric orientés CI/CD et approche “Code First”.  
J’espère que ces éléments pourront vous aider dans votre mise en œuvre.

---

# Microsoft Fabric CI/CD – Enterprise Code First Deployment Pattern
## Déploiement industrialisé de Notebooks, Variable Libraries et Data Pipelines avec fabric-cicd

---

# Contexte de la problématique

Le besoin exprimé est le suivant :

> Déployer en approche “Code First” des artefacts Microsoft Fabric via la librairie Python `fabric-cicd`, en permettant :
>
> 1. le déploiement d’un Notebook relié dynamiquement au bon Lakehouse selon le Workspace cible,
> 2. l’activation automatique du bon environnement dans une Variable Library,
> 3. le déploiement d’un Data Pipeline utilisant cette Variable Library,
> 4. le tout depuis une branche GitHub unique,
> 5. avec une stratégie CI/CD industrialisée et maintenable.

Ce document présente une implémentation Enterprise complète répondant précisément à ce scénario.

---

# Réponse courte à la question

Oui, ce scénario est totalement supporté par `fabric-cicd`, mais il faut distinguer 3 mécanismes différents :

| Sujet | Mécanisme |
|---|---|
| Binding Notebook ↔ Lakehouse | `parameter.yml` |
| Activation Variable Library | `environment=` dans `FabricWorkspace` |
| Références techniques Pipeline | `parameter.yml` |

C’est une distinction extrêmement importante dans Microsoft Fabric.

---

# Vision d’architecture recommandée

Le principe recommandé dans Fabric est :

> Un seul repository Git + une seule branche + promotion d’artefacts + injection dynamique des différences d’environnement.

Autrement dit :
- on ne duplique PAS les notebooks,
- on ne crée PAS un notebook DEV/TEST/PROD,
- on ne crée PAS plusieurs pipelines par environnement,
- on ne maintient PAS plusieurs branches Git.

Le moteur CI/CD doit promouvoir les mêmes artefacts entre environnements en injectant uniquement :
- les IDs,
- les bindings,
- les variables,
- les endpoints,
- les connexions.

---

# Architecture cible

```text
GitHub Repository
        │
        ▼
Azure DevOps / GitHub Actions
        │
        ▼
fabric-cicd Deployment Engine
        │
        ├── Workspace DEV
        │
        ├── Workspace TEST
        │
        └── Workspace PROD
```

---

# Structure Git recommandée

```text
fabric-project/
│
├── notebooks/
│   └── sales_notebook.Notebook/
│
├── pipelines/
│   └── ingest_pipeline.DataPipeline/
│
├── variableLibraries/
│   └── enterprise_variables.VariableLibrary/
│
├── semanticModels/
│   └── sales_model.SemanticModel/
│
├── deployment/
│   ├── parameter.yml
│   ├── deploy.py
│   ├── requirements.txt
│   └── deploy.yml
│
└── README.md
```

Cette structure est alignée avec :
- GitOps,
- DevOps,
- les patterns Fabric Enterprise,
- les pratiques Microsoft modernes.

---

# Étape 1 – Installation de fabric-cicd

## requirements.txt

```txt
fabric-cicd
pyyaml
```

---

# Installation Python

```bash
pip install fabric-cicd
```

---

# Étape 2 – Déploiement dynamique d’un Notebook relié au bon Lakehouse

---

# Pourquoi ce sujet est important

Dans Fabric, un Notebook contient des métadonnées spécifiques au Workspace.

Exemple :

```python
# META "default_lakehouse": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# META "default_lakehouse_workspace_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

Si ces GUIDs sont hardcodés :
- les déploiements deviennent fragiles,
- les notebooks deviennent dépendants d’un environnement,
- GitOps devient difficile à maintenir.

La bonne pratique Enterprise consiste donc à paramétrer dynamiquement ces métadonnées.

---

# Solution recommandée

Utiliser :
- `find_replace`,
- avec regex,
- dans `parameter.yml`.

---

# parameter.yml – Exemple Notebook

```yaml
find_replace:
  - find_value: '\#\s*META\s+"default_lakehouse":\s*"([0-9a-fA-F-]{36})"'
    replace_value:
      DEV: "$items.Lakehouse.LH_BRONZE.id"
      TEST: "$items.Lakehouse.LH_BRONZE.id"
      PROD: "$items.Lakehouse.LH_BRONZE.id"
    is_regex: "true"
    item_type: "Notebook"
    item_name: ["Sales Notebook"]

  - find_value: '\#\s*META\s+"default_lakehouse_workspace_id":\s*"([0-9a-fA-F-]{36})"'
    replace_value:
      DEV: "$workspace.id"
      TEST: "$workspace.id"
      PROD: "$workspace.id"
    is_regex: "true"
    item_type: "Notebook"
    item_name: ["Sales Notebook"]
```

---

# Résultat obtenu

Au moment du déploiement :
- le Notebook est automatiquement relié au bon Lakehouse,
- le Workspace ID est automatiquement injecté,
- le même Notebook fonctionne dans tous les environnements.

Cela permet :
- une vraie stratégie GitOps,
- zéro duplication,
- un déploiement industrialisé.

---

# Étape 3 – Activation automatique de la Variable Library

---

# Point extrêmement important

Le value set actif de la Variable Library n’est PAS piloté par `parameter.yml`.

Dans `fabric-cicd`, le mécanisme officiel repose sur :

```python
environment="DEV|TEST|PROD"
```

dans l’objet `FabricWorkspace`.

C’est LE point clé de votre question.

---

# deploy.py – Exemple complet

```python
from fabric_cicd import FabricWorkspace, publish_all_items

workspace_id = "YOUR_WORKSPACE_ID"

target_workspace = FabricWorkspace(
    workspace_id=workspace_id,
    repository_directory=".",
    item_type_in_scope=[
        "Lakehouse",
        "Notebook",
        "VariableLibrary",
        "DataPipeline",
        "SemanticModel"
    ],
    environment="TEST"
)

publish_all_items(target_workspace)
```

---

# Ce que fait réellement `environment="TEST"`

Lors du déploiement :
- la Variable Library est publiée,
- le value set `TEST` devient automatiquement actif,
- les Pipelines et Notebooks consomment automatiquement les bonnes valeurs.

Cela évite :
- les variables hardcodées,
- les Variable Libraries dupliquées,
- les branches Git multiples.

---

# Étape 4 – Déploiement du Data Pipeline

---

# Cas 1 – Références internes Fabric

Si le Pipeline utilise :
- des Lakehouses,
- des Notebooks,
- des Warehouses,
- des artefacts Fabric natifs,

alors `fabric-cicd` sait généralement repointer automatiquement les références.

Très peu de paramétrage est nécessaire.

---

# Cas 2 – Références externes

Pour :
- les APIs,
- les endpoints,
- les Connection IDs,
- les linked services,
- les gateways,
- les services externes,

il faut utiliser `parameter.yml`.

---

# parameter.yml – Exemple Pipeline

```yaml
key_value_replace:
  - find_key: $.properties.activities[?(@.name=="Copy Data")].typeProperties.source.datasetSettings.externalReferences.connection
    replace_value:
      DEV: "connection-id-dev"
      TEST: "connection-id-test"
      PROD: "connection-id-prod"
    item_type: "DataPipeline"
    item_name: "Ingest Pipeline"
```

---

# Point important sur les connexions Fabric

Actuellement :
- les Connections Fabric ne sont PAS source-controlled,
- elles doivent déjà exister dans le Workspace cible,
- l’identité CI/CD doit déjà avoir les permissions nécessaires.

C’est une limitation actuelle importante de Fabric.

---

# Étape 5 – Paramétrisation des Semantic Models

Votre exemple avec :

```yaml
find_replace:
```

est totalement correct.

Exemple :

```yaml
find_replace:
  - find_value: "URL_Fichier"
    replace_value:
      DEV: "https://dev.sharepoint.com/file.json"
      TEST: "https://test.sharepoint.com/file.json"
      PROD: "https://prod.sharepoint.com/file.json"
    item_type: "SemanticModel"
```

Cette approche est parfaitement adaptée :
- aux APIs,
- aux JSON,
- aux endpoints SharePoint,
- aux sources Power Query.

---

# Point d’attention SharePoint / JSON

Le cas suivant est très fréquent :

```text
HTTP/1.1 200 OK
```

mais sans retour réel du JSON attendu.

Dans Fabric cela signifie généralement :
- une redirection HTML SharePoint,
- un problème d’authentification,
- des permissions insuffisantes,
- ou une URL non directe.

Je recommande donc toujours de vérifier :
- le content-type,
- les permissions du Service Principal,
- la Managed Identity,
- et l’accessibilité directe du fichier.

---

# Recommandation Enterprise Microsoft Fabric

Pour les projets Enterprise Fabric, je recommande fortement :

| Sujet | Recommandation |
|---|---|
| Branching | Une seule branche Git |
| Déploiement | Promotion d’artefacts |
| Variables | Variable Library |
| Secrets | Azure Key Vault |
| IDs techniques | parameter.yml |
| Notebook binding | Regex parameterization |
| CI/CD | GitHub Actions ou Azure DevOps |
| Gouvernance | Microsoft Purview |

---

# Exemple GitHub Actions

## .github/workflows/deploy.yml

```yaml
name: Fabric Deployment

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r deployment/requirements.txt

      - name: Deploy Fabric Artifacts
        run: |
          python deployment/deploy.py
```

---

# Bonnes pratiques Enterprise

---

# 1. Ne jamais hardcoder les IDs

Toujours paramétrer :
- Workspace IDs,
- Lakehouse IDs,
- Connection IDs,
- endpoints,
- APIs.

---

# 2. Utiliser une stratégie mono-branche

Éviter :
- branche DEV,
- branche TEST,
- branche PROD.

Préférer :
- UNE branche,
- promotion automatisée.

---

# 3. Externaliser les secrets

Ne jamais stocker :
- passwords,
- tokens,
- secrets,
- credentials

dans Git.

Préférer :
- Azure Key Vault,
- Managed Identity.

---

# 4. Standardiser les déploiements

Maintenir :
- un seul moteur de déploiement,
- un seul parameter.yml,
- un seul process CI/CD.

---

# Conclusion

Oui, votre scénario est totalement réalisable avec `fabric-cicd`.

La bonne architecture consiste à :
- paramétrer les bindings Notebook via `parameter.yml`,
- piloter l’environnement Variable Library via `environment=`,
- paramétrer les références techniques Pipeline via `parameter.yml`,
- promouvoir les mêmes artefacts entre environnements.

C’est aujourd’hui l’approche la plus propre et la plus scalable pour industrialiser Microsoft Fabric dans un contexte Enterprise.

---

# Ressources utiles

- fabric-cicd GitHub Repository
- Microsoft Fabric CI/CD documentation
- Microsoft Fabric Deployment Pipelines
- Documentation Variable Libraries

---

# Fin du document
