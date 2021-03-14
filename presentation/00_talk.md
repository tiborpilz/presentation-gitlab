---
title: Gitlab CI/CD
author: Tibor Pilz
date: 15.03.2021
---

# Overview

- Intro
- Jobs
- Pipelines
- Variablen
- Include/Extends
- Carboncopy

---

# Intro

- Gitlab CI/CD führt Aktionen aus
- Konfiguriert durch `gitlab-ci.yml`
- Konfiguration beschreibt eine Pipeline
- Eine Pipeline besteht aus einem oder mehreren Jobs
- Jobs werden in Stages eingeteilt
- Stages bestimmen die Reihenfolge der Jobs

---

# Job

```{.yaml}
hello_world_job:
  script:
    - echo "Hallo, Welt!"
```

- Kleinste Einheit einer Pipeline
- Der Name darf nicht einem Gitlab-Keyword entsprechen
- Script beschreibt die Aktion

::: notes

Der Exit-Code des Shellscripts bestimmt, ob der Job fehlschlägt oder erfolgreich ist.
Sofern nicht anders konfiguriert, stoppen fehlgeschlagene Scripts die Pipeline und lassen sie fehlschlagen.

:::

---

## Job - Ausführung

- Gitlab Runner arbeitet die Jobs ab
- Jobs werden per Docker-Container ausgeführt
- `script` läuft innerhalb des Repositories
- Zusätzlich zu `script` kann `before_script` und `after_script` benutzt werden

::: notes



:::

---

## Docker

```{.yaml}
node_job:
  image: node:12
  script:
    - npm install
    - npm run test
```

- Jobs werden (meistens) per Docker-Container ausgeführt
- Wird kein Image angegeben, benutzt Gitlab das Instanzenweite Fallback-Image

::: notes

Auf code.avenga.cloud wird Standardmässig `avenga/gitlab-job` benutzt

::: 

---

## Container - DinD

- Da Jobs innerhalb eines Containers ausgeführt werden, ist `dind` notwendig

```{.yaml}
build_docker:
  image: docker:dind
  script: docker build .
```

---

## Variablen

- Predefined:
  - Von Gitlab vorgegeben
  - Informationen über die Pipeline, das Projekt, das Repository, etc.
- Userdefined:
  - In der UI oder der Config definierbar

::: notes

In Gitlab gibt es Variablen, die entweder Userdefiniert, oder von Gitlab vorgegeben sind.
Userdefinierte Variablen können über die Config oder pro Projekt/Gruppe/Instanz per UI definiert werden.
Von Gitlab vorgegebene Variablen liefern u.A. Informationen über das Projekt, in dem die Pipeline ausgeführt wird.

:::

---

## Variablen - Userdefiniert


```{.yaml data-line-numbers=[6|8]}
variables:
  NAME: "Welt"

greeting_job:
  variables:
    GREETING: "Hallo"
  script: echo $GREETING, $NAME!
```


::: notes

`variables` ist ein Keyword und damit ein Beispiel für einen ungültigen Job-Namen

:::

## Variablen - Prädefiniert

Beispiele:
  - `CI_COMMIT_SHA` - SHA des Commits, für den die Pipeline ausgeführt wird.
  - `CI_PROJECT_TITLE` - Titel des Projekts
  - `CI_REGISTRY_IMAGE` - Image des Projekts in der Gitlab Container Registry
---

## Variablen - Anwendung

- Variablen sind nicht nur im `script` anwendbar
- In der Praxis ist es sinnvoll, prädefinierte und userdefinierte Variablen zu verbinden
- Beispiel: dynamisches Image

```{.yaml}
variables:
  HELPER_IMAGE: $CI_REGISTRY_IMAGE/helper

dynamic_image_job:
  image: $HELPER_IMAGE
  ...
```

---

## Regeln

- Die Ausführung von Jobs kann durch `rules` eingeschränkt werden

---

## Stages

- Um die Reihenfolge der Jobs festzulegen, sind Stages notwendig
- Stages werden als Top-Level Array definiert
- Die Zugehörigkeit der Jobs wird per `stage`-Keyword angegeben
- Es werden alle Jobs einer Stage ausgeführt, bevor die nächste Stage angefangen wird

---

## Stages

```{.yaml data-line-numbers=[|1-3|5-15|2,6,10|3,15|]}
stages:
  - test
  - build

test_unit:
  stage: test
  script: echo "running unit tests"

test_integration:
  stage: test
  script: echo "running integration tests"

build:
  stage: build
  script: echo "building the thing"
```

::: notes

`stages` ist ein Keyword und damit ein Beispiel für einen ungültigen Job-Namen

:::

---

## Stages - Visualisierung

![Test und Build Stages](images/stages.png "Test und Build Stages")

---

## Stages - Mehrere Services 

```{.yaml data-line-numbers=[|2,5,12|3,16,20|5,16|12,20|]}
stages:
  - build
  - deploy

build_foo:
  stage: build
  script: "building foo"
  script:
    - echo "build foo"
    - sleep 30

build_bar:
  stage: build
  script: "building bar"

deploy_foo:
  stage: deploy
  script: "deploy foo"

deploy_bar:
  stage: deploy
  script: "deploy bar"
```

::: notes

Das Beispiel beschreibt zwei Services, `foo` und `bar`.
Die Einteilung in Stages lässt zuerst beide Bauprozesse stattfinden, dann beide Deployprozesse. Also muss `bar` warten, bis `foo` fertig gebaut ist.

:::

---

![Stage-Pipeline für Services `foo` und `bar`](images/stages-services.png)

---

## DAG

- Directed Acyclic Graph
- Alternative / Ergänzung zu Stages
- Beschreibt für jeden Job die Jobs, die vorher ausgeführt werden müssen
- Wird per `needs` in den Jobs angegeben

---

## DAG Beispiel

```{.yaml data-line-numbers=[|18,23|]}
stages:
  - build
  - deploy

build_foo:
  stage: build
  script:
    - echo "build foo"
    - sleep 30

build_bar:
  stage: build
  script: echo "build bar"

deploy_foo:
  stage: deploy
  script: echo "deploy foo"
  needs: [build_foo]

deploy_bar:
  stage: deploy
  script: echo "deploy bar"
  needs: [build_bar]
```

::: notes

:::

---

## DAG Visualisierung

![DAG Pipeline](images/DAG-pipeline.png)

---

## DAG Visualisierung

![DAG Needs](images/DAG-needs.png)

---


---

### Presets
### Priorität
### Anwendung

---

# Extends

## Hidden Jobs

---

# Include

---

# Rules

## if
## changes
## when
---

## DAGs
