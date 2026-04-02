# README CI/CD – POC DevSecOps

Ce document décrit le pipeline GitLab CI implémenté dans `.gitlab-ci.yml` pour l'application Full-Stack Angular + Spring Boot.

## 1) Vue d'ensemble

Le pipeline est structuré en 5 stages :

1. `lint`
   - `lint:frontend` : ESLint sur le code Angular/TypeScript.
   - `spotbugs:backend` : analyse statique Java avec SpotBugs + FindSecBugs.
2. `test`
   - `test:frontend` : tests unitaires Karma (Chrome Headless, sans watch) + couverture.
   - `test:backend` : tests unitaires JUnit + rapport Jacoco.
3. `quality`
   - stage réservé pour extensions futures (ex: SonarQube Quality Gate).
4. `package`
   - Build des 3 images Docker (`front`, `back`, `standalone`) depuis le Dockerfile multi-stage.
   - Scan Trivy (échec si vulnérabilité `CRITICAL`).
   - Push vers GitLab Container Registry privé.
5. `deploy`
   - `deploy:staging` manuel, via `docker compose`.

## 2) Lint & analyse statique

## Frontend (ESLint)

- Fichiers ajoutés :
  - `front/.eslintrc.json`
  - `front/.eslintignore`
- Script npm ajouté : `npm run lint`.

Le job `lint:frontend` exécute :

```bash
cd front
npm install
npm run lint
```

## Backend (SpotBugs)

Le fichier `back/build.gradle` inclut :

- plugin `com.github.spotbugs`
- plugin `jacoco`
- dépendance `findsecbugs-plugin` (détection orientée sécurité)

Le job `spotbugs:backend` exécute :

```bash
cd back
./gradlew spotbugsMain spotbugsTest
```

Rapports publiés en artifacts :

- `back/build/reports/spotbugs/`

## 3) Tests automatisés + couverture

## Frontend

- Script CI ajouté : `npm run test:ci`
- `karma.conf.js` configuré pour produire :
  - HTML
  - text-summary
  - lcov
  - cobertura

Commande exécutée :

```bash
CHROME_BIN=/usr/bin/google-chrome npm run test:ci
```

Artifacts :

- `front/coverage/`

## Backend

Commande exécutée :

```bash
./gradlew test jacocoTestReport
```

Artifacts / rapports :

- JUnit XML : `back/build/test-results/test/*.xml`
- Jacoco HTML/XML : `back/build/reports/jacoco/test/`

## 4) Build Docker + versionning

Les images sont construites depuis le `Dockerfile` racine avec les targets :

- `front`  -> image `frontend`
- `back` -> image `backend`
- `standalone` -> image `all-in-one`

Tag d'image :

- si pipeline sur tag Git : `CI_COMMIT_TAG`
- sinon : `CI_COMMIT_SHORT_SHA`

Cela permet un versionning sémantique (release tag) ou technique (SHA).

## 5) Scan sécurité Trivy

Dans chaque job de packaging :

1. Build image locale
2. Scan Trivy avec sévérité `CRITICAL`
3. Échec du job si CVE critique détectée (`--exit-code 1`)
4. Push de l'image si scan OK

Exemple de commande :

```bash
trivy image --no-progress --severity CRITICAL --exit-code 1 <image:tag>
```

## 6) Publication registry interne

Le pipeline pousse vers le registry GitLab natif :

- `$CI_REGISTRY_IMAGE/frontend:$IMAGE_TAG`
- `$CI_REGISTRY_IMAGE/backend:$IMAGE_TAG`
- `$CI_REGISTRY_IMAGE/all-in-one:$IMAGE_TAG`

Auth via variables GitLab :

- `CI_REGISTRY`
- `CI_REGISTRY_USER`
- `CI_REGISTRY_PASSWORD`

Optionnel : sur la branche par défaut, tag `latest` également publié.

## 7) Déploiement staging (manuel)

Fichier fourni : `docker-compose.yml`

Le job `deploy:staging` :

- injecte les variables d'image (`FRONTEND_IMAGE`, `BACKEND_IMAGE`, `ALL_IN_ONE_IMAGE`)
- exécute `docker compose pull`
- exécute `docker compose up -d`

Ce job est `manual` pour garder le contrôle sur la promotion vers staging.

## 8) Variables GitLab recommandées

À configurer dans **Settings > CI/CD > Variables** :

- `CI_REGISTRY`, `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD` (souvent déjà fournis par GitLab)
- (optionnel) variables de déploiement staging selon l'infra cible (SSH, hôte, etc.)

## 9) Points d'amélioration possibles

- Ajouter SonarQube + Quality Gate bloquante (backend + frontend).
- Ajouter SAST/Dependency Scanning GitLab natifs.
- Signer les images (Cosign) et générer SBOM (CycloneDX/SPDX).
- Ajouter promotion multi-env (staging -> prod) avec approbation.
