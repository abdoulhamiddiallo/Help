# Industrialisation des déploiements Microsoft Fabric
## Architecture CI/CD Enterprise – Data Pipelines & User Data Functions

---

# 1. Objectif

Mettre en place une architecture de déploiement Microsoft Fabric robuste permettant :

- le déploiement automatisé des artifacts Fabric ;
- la gestion multi-environnements ;
- le remplacement dynamique des connexions ;
- le support des :
  - Data Pipelines ;
  - User Data Functions Python ;
  - autres artifacts Fabric à terme ;
- l’industrialisation CI/CD via GitHub Actions.

---

# 2. Analyse de l’approche initiale

L’approche initiale était globalement bonne :

- séparation TEST / PROD ;
- workflows GitHub Actions ;
- script Python de déploiement ;
- paramétrage YAML ;
- remplacement dynamique des connexions.

Cependant plusieurs problèmes techniques devaient être corrigés afin d’obtenir une architecture réellement enterprise-ready.

---

# 3. Principales erreurs identifiées

## 3.1 User Data Functions ≠ Data Pipelines

### Problème

L’approche initiale supposait que les User Data Functions utilisaient le même modèle de connexion que les Data Pipelines.

Exemple incorrect :

```yaml
$.properties.activities[...].externalReferences.connection
```

### Réalité Fabric

Les User Data Functions utilisent généralement :

```json
connectedDataSources
```

Exemple :

```json
{
  "connectedDataSources": [
    {
      "alias": "lakehouse",
      "artifactId": "lakehouse-id",
      "workspaceId": "workspace-id"
    }
  ]
}
```

### Correction

Le moteur de remplacement doit cibler :

```yaml
$.connectedDataSources[?(@.alias=="lakehouse")].artifactId
```

et :

```yaml
$.connectedDataSources[?(@.alias=="lakehouse")].workspaceId
```

---

## 3.2 Mutation permanente des fichiers Git

### Problème

Le script modifiait directement les artifacts du repository Git.

### Correction

Toujours utiliser un dossier temporaire avant déploiement.

```python
import shutil
import tempfile

temp_dir = tempfile.mkdtemp()
```

Puis :

```python
shutil.copytree(
    source_artifacts,
    temp_artifacts,
    dirs_exist_ok=True
)
```

Les remplacements sont appliqués uniquement dans le dossier temporaire.

---

## 3.3 Absence de gestion des dépendances

### Correction

Définir explicitement l’ordre de déploiement :

```python
DEPLOYMENT_ORDER = [
    "Environment",
    "Lakehouse",
    "Notebook",
    "DataPipeline",
    "UserDataFunction",
    "SemanticModel"
]
```

---

## 3.4 Sécurité insuffisante

### Correction recommandée

Utiliser :

- GitHub OIDC ;
- Azure Federated Identity ;
- Azure Login OIDC.

---

## 3.5 Absence de rollback

### Correction

Stocker :
- définition précédente ;
- version ;
- commit SHA ;
- état du déploiement.

Exemple :

```json
{
  "deploymentId": "DEP-001",
  "environment": "PROD",
  "previousVersion": "v1.2.4"
}
```

---

## 3.6 Absence de validation

### Correction

Ajouter :

```python
from jsonschema import validate
```

et des vérifications Fabric :
- workspace ;
- connexion ;
- lakehouse ;
- runtime.

---

# 4. Architecture corrigée

```text
.github/
  workflows/
    deploy-test.yml
    deploy-prod.yml

fabric/

  deploy.py

  core/
    validation.py
    replacements.py
    deployment.py
    rollback.py

  configs/
    parameter.yml

  artifacts/
```

---

# 5. Paramétrage des environnements

## configs/parameter.yml

```yaml
environments:

  TEST:

    workspace: fabric-test

    replacements:

      - artifact_type: DataPipeline
        artifact_name: "Ingest Pipeline"

        file: pipeline-content.json

        json_path: '$.properties.activities[?(@.name=="Copy Data")].typeProperties.source.datasetSettings.externalReferences.connection'

        value: connection-test

      - artifact_type: UserDataFunction
        artifact_name: "Customer Scoring"

        file: definition.json

        json_path: '$.connectedDataSources[?(@.alias=="lakehouse")].artifactId'

        value: lakehouse-test-id
```

---

# 6. Moteur de remplacement corrigé

## core/replacements.py

```python
from pathlib import Path
import json

from jsonpath_ng.ext import parse


def load_json(path):

    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)


def save_json(path, payload):

    with open(path, "w", encoding="utf-8") as f:
        json.dump(payload, f, indent=2)


def apply_jsonpath_replace(
    json_payload,
    json_path,
    value
):

    expression = parse(json_path)

    matches = expression.find(json_payload)

    if not matches:
        raise ValueError(
            f"Aucun match trouvé : {json_path}"
        )

    for match in matches:

        match.full_path.update(
            json_payload,
            value
        )

    return json_payload
```

---

# 7. Gestion correcte du dossier temporaire

```python
import shutil
import tempfile

from pathlib import Path


def prepare_temp_artifacts(source_dir):

    temp_root = tempfile.mkdtemp()

    temp_artifacts = (
        Path(temp_root)
        / "artifacts"
    )

    shutil.copytree(
        source_dir,
        temp_artifacts,
        dirs_exist_ok=True
    )

    return temp_artifacts
```

---

# 8. Ordonnancement des déploiements

```python
DEPLOYMENT_ORDER = [
    "Environment",
    "Lakehouse",
    "Notebook",
    "DataPipeline",
    "UserDataFunction",
    "SemanticModel"
]
```

---

# 9. GitHub Actions corrigé

## .github/workflows/deploy-test.yml

```yaml
name: Deploy TEST

on:
  push:
    branches:
      - main

jobs:

  deploy:

    runs-on: ubuntu-latest

    environment: TEST

    permissions:
      id-token: write
      contents: read

    steps:

      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5

        with:
          python-version: "3.11"

      - run: |
          pip install -r requirements.txt

      - name: Azure Login OIDC
        uses: azure/login@v2

        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy TEST
        run: |
          python fabric/deploy.py TEST
```

---

# 10. Bonnes pratiques enterprise

Toujours :

- utiliser GitHub Environments ;
- séparer TEST et PROD ;
- utiliser OIDC ;
- externaliser les paramètres ;
- versionner tous les artifacts ;
- tracer les déploiements ;
- prévoir rollback et validation.

---

# 11. Conclusion

Cette version corrigée apporte :

- une meilleure gouvernance ;
- une sécurité moderne ;
- une compatibilité Fabric réelle ;
- une architecture extensible ;
- une industrialisation CI/CD crédible.

Elle constitue désormais une base sérieuse pour un projet Microsoft Fabric enterprise.
