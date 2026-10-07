# ci-templates — workflows GitHub Actions partagés

Ce dépôt sert à **centraliser la CI/CD** de tous mes projets : au lieu de recopier les mêmes
étapes dans chaque dépôt, chaque projet contient seulement de petits fichiers « appelants »
qui réutilisent les workflows d'ici.

- **CI** (intégration continue) : à chaque pull request et à chaque push sur `main`, on installe
  les dépendances, on lance le lint, les tests et le build, et on vérifie que les images Docker se construisent.
- **CD** en mode *Continuous Delivery* (livraison continue) : la livraison (publication des images
  Docker ou d'un paquet d'artefacts) est toujours prête, mais **ne part qu'après une validation manuelle**.

Ce dépôt ne contient **aucun secret ni code de projet** : uniquement de la logique générique.

## Utilisation

Dans un projet, on appelle un workflow avec `uses: Ramcy-cloud/ci-templates/.github/workflows/<fichier>@v1`.
Le tag `v1` est stable : les corrections compatibles y sont publiées, les changements cassants iront dans `v2`.

| Workflow | Rôle |
| --- | --- |
| [`python-ci.yml`](.github/workflows/python-ci.yml) | Installe un projet Python, lance le lint (optionnel) et les tests |
| [`node-ci.yml`](.github/workflows/node-ci.yml) | `npm ci`, puis lint / tests / build (chacun optionnel) |
| [`docker-build-check.yml`](.github/workflows/docker-build-check.yml) | Construit les images Docker **sans les publier** (vérifie les Dockerfiles) |
| [`docker-delivery.yml`](.github/workflows/docker-delivery.yml) | Construit et **publie** les images sur `ghcr.io`, derrière un environnement GitHub |
| [`artifact-delivery.yml`](.github/workflows/artifact-delivery.yml) | Livraison sans Docker : republie les artefacts de build, derrière un environnement GitHub |

---

### `python-ci.yml`

| Entrée | Défaut | Description |
| --- | --- | --- |
| `working-directory` | `.` | Dossier du projet Python |
| `python-version` | `3.11` | Version de Python |
| `requirements-file` | `requirements.txt` | Fichier de dépendances (relatif à `working-directory`, vide = aucun) |
| `install-command` | – | Installation supplémentaire |
| `lint-command` | – | Commande de lint (ex. `ruff check .`) |
| `lint-allow-failure` | `false` | `true` = un lint en échec donne un avertissement visible sans bloquer (dette existante à résorber) |
| `setup-command` | – | Préparation avant les tests (ex. indexer un corpus) |
| `test-command` | `pytest` | Commande de test |
| `env` | – | Variables **non secrètes**, une par ligne : `CLE=valeur` |
| `artifact-name` / `artifact-path` | – | Publie un artefact après les tests (chemins relatifs à la racine du dépôt) |
| `artifact-retention-days` | `7` | Conservation de l'artefact |

Le cache pip est activé automatiquement (clé : `requirements*.txt` du dossier).

```yaml
jobs:
  backend:
    uses: Ramcy-cloud/ci-templates/.github/workflows/python-ci.yml@v1
    with:
      working-directory: backend
      python-version: "3.11"
      test-command: pytest -q
```

### `node-ci.yml`

| Entrée | Défaut | Description |
| --- | --- | --- |
| `working-directory` | `.` | Dossier qui contient `package.json` et `package-lock.json` |
| `node-version` | `22` | Version de Node (Vite 8 exige 20.19+ ou 22.12+) |
| `install-command` | `npm ci` | Installation des dépendances (cache npm activé) |
| `lint-command` / `test-command` / `build-command` | – | Étapes optionnelles, lancées dans cet ordre |
| `lint-allow-failure` | `false` | `true` = un lint en échec donne un avertissement visible sans bloquer (dette existante à résorber) |
| `env` | – | Variables **non secrètes**, une par ligne : `CLE=valeur` |
| `artifact-name` / `artifact-path` | – | Publie un artefact après le build (chemins relatifs à la racine du dépôt) |
| `artifact-retention-days` | `7` | Conservation de l'artefact |

```yaml
jobs:
  frontend:
    uses: Ramcy-cloud/ci-templates/.github/workflows/node-ci.yml@v1
    with:
      working-directory: frontend
      lint-command: npm run lint
      build-command: npm run build
```

### `docker-build-check.yml`

| Entrée | Défaut | Description |
| --- | --- | --- |
| `images` | (obligatoire) | Liste JSON : `[{"name": "backend", "context": "backend", "dockerfile": "backend/Dockerfile"}]` (`dockerfile` optionnel) |
| `free-disk-space` | `false` | Libère de l'espace disque sur le runner (images lourdes : torch…) |

```yaml
jobs:
  docker:
    uses: Ramcy-cloud/ci-templates/.github/workflows/docker-build-check.yml@v1
    with:
      images: '[{"name": "backend", "context": "backend"}, {"name": "frontend", "context": "frontend"}]'
```

### `docker-delivery.yml`

Publie chaque image sous `ghcr.io/<propriétaire>/<dépôt><separator><name>`, **converti en minuscules**
(Docker refuse les majuscules), avec deux tags : le **sha du commit** et `latest`.

| Entrée | Défaut | Description |
| --- | --- | --- |
| `images` | (obligatoire) | Même format que `docker-build-check.yml` |
| `environment` | `production` | Environnement GitHub du job : c'est lui qui porte la validation manuelle |
| `ref` | commit déclencheur | Commit à livrer (utile avec `workflow_run`) |
| `separator` | `/` | `/` → `ghcr.io/owner/depot/backend`, `-` → `ghcr.io/owner/depot-backend` |
| `free-disk-space` | `false` | Libère de l'espace disque sur le runner |

Le job appelant doit accorder `packages: write`.

```yaml
jobs:
  livraison:
    permissions:
      contents: read
      packages: write
    uses: Ramcy-cloud/ci-templates/.github/workflows/docker-delivery.yml@v1
    with:
      images: '[{"name": "backend", "context": "backend"}]'
      environment: production
```

### `artifact-delivery.yml`

Pour les projets **sans Dockerfile** : récupère les artefacts de build produits par la CI
(`artifact-name` de `python-ci` / `node-ci`) et les republie en un paquet `livraison-<sha>`, conservé 90 jours.

| Entrée | Défaut | Description |
| --- | --- | --- |
| `pattern` | `build-*` | Artefacts à livrer |
| `run-id` | exécution courante | Exécution qui a produit les artefacts (ex. `${{ github.event.workflow_run.id }}`) |
| `sha` | `github.sha` | Commit livré (nom du paquet) |
| `environment` | `production` | Environnement GitHub du job (validation manuelle) |
| `retention-days` | `90` | Conservation du paquet livré |

Le job appelant doit accorder `actions: read`.

---

## Choisir le mode de validation manuelle

**Dépôt public** — environnement avec validation :

1. Créer l'environnement `production` (Settings → Environments) avec *Required reviewers*.
2. Le `cd.yml` du projet se déclenche après une CI verte sur `main` (`on: workflow_run`).
3. Le job de livraison attend : on clique sur **Review deployments → Approve** dans l'onglet *Actions*.

```yaml
on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]
jobs:
  livraison:
    if: github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'push'
    permissions:
      contents: read
      packages: write
    uses: Ramcy-cloud/ci-templates/.github/workflows/docker-delivery.yml@v1
    with:
      images: '[{"name": "backend", "context": "backend"}]'
      ref: ${{ github.event.workflow_run.head_sha }}
```

**Dépôt privé sur un compte gratuit** — les *Required reviewers* n'y sont pas disponibles :
le `cd.yml` se déclenche **à la main** (`on: workflow_dispatch`, bouton **Run workflow** dans l'onglet *Actions*),
relance la CI puis livre. Le clic sur le bouton est la validation.

```yaml
on:
  workflow_dispatch:
jobs:
  tests:
    uses: Ramcy-cloud/ci-templates/.github/workflows/python-ci.yml@v1
    with:
      working-directory: backend
  livraison:
    needs: [tests]
    permissions:
      contents: read
      packages: write
    uses: Ramcy-cloud/ci-templates/.github/workflows/docker-delivery.yml@v1
    with:
      images: '[{"name": "backend", "context": "backend"}]'
```

## Versions des actions utilisées

`actions/checkout@v7`, `actions/setup-python@v7`, `actions/setup-node@v7`, `actions/upload-artifact@v7`,
`actions/download-artifact@v8`, `docker/setup-buildx-action@v4`, `docker/login-action@v4`, `docker/build-push-action@v7`.
Chaque workflow déclare des `permissions` minimales (`contents: read`, plus `packages: write` ou `actions: read` uniquement là où c'est nécessaire).
