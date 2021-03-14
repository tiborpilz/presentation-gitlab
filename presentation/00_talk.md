---
title: Gitlab CI/CD
author: Tibor Pilz
date: 15.03.2021
---

# Overview

- Intro
- Jobs
- Pipelines
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

## Job - `image`

```{.yaml data-line-numbers=[|2|]}
node_job:
  image: node:12
  script:
    - npm install
    - npm run test
```

- `image` gibt das Docker-Image an
- Wird kein Image angegeben, benutzt Gitlab das Instanzenweite Fallback-Image

::: notes

Auf code.avenga.cloud wird Standardmässig `avenga/gitlab-job` benutzt

::: 

---

## Job - Docker in Docker

```{.yaml}
build_docker:
  image: docker:dind
  script: docker build .
```

- Um Docker innerhalb eines Jobs zu nutzen, ist Docker-in-Docker nötig
- Erweiterte Docker-Features wie `volumes` oder `networks` haben Fallstricke
- Gitlab hat eine Container-Registry

::: notes

Einige Fallstricke bei der Ausführung, besonders wenn ein Volume gemounted werden, oder per `docker-compose` Netzwerke erstellt werden sollen.

:::

---

## Variablen

- Vordefiniert
  - Von Gitlab vorgegeben
  - Informationen über die Pipeline, das Projekt, das Repository, etc.
- Userdefiniert:
  - In der UI oder der Config definierbar

---

## Variablen - Userdefiniert


```{.yaml data-line-numbers=[|2,6|7|]}
variables:
  GREETING: "Hallo"

greeting_job:
  variables:
    NAME: "Welt"
  script: echo $GREETING, $NAME!
```

::: notes

`variables` ist ein Keyword und damit ein Beispiel für einen ungültigen Job-Namen

:::

---

## Variablen - Vordefiniert

Beispiele:

- `CI_COMMIT_SHA`
- `CI_PROJECT_TITLE`
- `CI_REGISTRY_IMAGE`

::: notes

- SHA des Commits, für den die Pipeline ausgeführt wird.
- Titel des Projekts
- Image des Projekts in der Gitlab Container Registry

Es gibt sehr viele Vordefinierte Variablen, es lohnt sich hier besonders, oft in die Docs zu schauen

:::

---

## Variablen - Anwendung

- Variablen sind nicht nur im `script` anwendbar
- In der Praxis ist es sinnvoll, prädefinierte und userdefinierte Variablen zu verbinden
- Beispiel: dynamisches Image

```{.yaml data-line-numbers=[|2|5|]}
variables:
  HELPER_IMAGE: $CI_REGISTRY_IMAGE/helper

dynamic_image_job:
  image: $HELPER_IMAGE
  script: ...
```

---

## Job - `rules`

- Die Ausführung von Jobs kann durch `rules` eingeschränkt werden
- Ein Job kann mehrere `rules` haben
- `if` evaluiert ein Statement
- `changes` überprüft, ob Dateien verändert wurden
- `exists` überprüft, ob Dateien existieren

---

## Job - `rules`

```{.yaml}
rules_job:
  script: echo "Hello, rules!"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - service_a/**
```

::: notes

Wird nur ausgeführt, wenn Änderungen im `service_a` Directory sind *und* die Pipeline für einen Merge Request ausgeführt wird.

:::

---

## Stages

- Bestimmen die Reihenfolge der Jobs
- Werden als Top-Level Array definiert
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

```{.yaml data-line-numbers=[|1-3|5-14|16-22|5-10,16-18|12-14,20-22|]}
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
- Wird per `needs` in den Jobs angegeben
- Beschreibt für jeden Job die Jobs, die vorher ausgeführt werden müssen

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

## Config Strukturieren

- Extends
  - "erbt" einen bestehenden Job
- Include
  - Import von Gitlab-Config Dateien

---

## Extends

```{.yaml}
.echo_before:
  before_script: echo "Starting the job"
  
extends_job:
  extends: .echo_before
  script: echo "Doing the job"
```

## Hidden Jobs

---

# Include

---
