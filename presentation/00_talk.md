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

# Jobs

```{.yaml}
hello_world_job:
  script:
    - echo "Hallo, Welt!"
```

- Kleinste Einheit einer Pipeline
- Script beschreibt die Aktion

::: notes

- Ein Job ist die kleinste Einheit einer Pipeline
- Job ist ein Top-Level Objekt
- Bis auf Ausnahmen (Keywords) wird der Key frei gewählt und ist der Name des Jobs
- Die Ausnahmen sind Gitlab Keywords
- Script ist notwendig, beschreibt die Aktion, die ausgeführt werden soll
Eine oder mehrere Zeilen Shellscript.
Der Exit-Code des Shellscripts bestimmt, ob der Job fehlschlägt oder erfolgreich ist.
Sofern nicht anders konfiguriert, stoppen fehlgeschlagene Scripts die Pipeline und lassen sie fehlschlagen.

:::

---

## Jobs - Image

- Jobs werden per Container ausgeführt
- Das Container-Image kann pro Pipeline oder pro Job angegeben werden

```{.yaml data-line-numbers=[2|4,5]}
node_job:
  image: node:12
  script:
    - npm install
    - npm run test
```

::: notes

Das Image muss nicht angegeben werden
Standardmässig wird avenga/gitlab-job benutzt

::: 

---

## Stages

- Um die Reihenfolge der Jobs festzulegen, sind Stages notwendig
- Stages werden als Top-Level Array definiert
- Die Zugehörigkeit der Jobs wird per `stage`-Keyword angegeben
- Es werden alle Jobs einer Stage ausgeführt, bevor die nächste Stage angefangen wird

---

## Stages

```{.yaml data-line-numbers=[1-18|1-3|5-18|2,6,11|3,16]}
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

## Stages

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

## Variablen

In Gitlab gibt es Variablen, die entweder Userdefiniert, oder von Gitlab vorgegeben sind.
Userdefinierte Variablen können über die Config oder pro Projekt/Gruppe/Instanz per UI definiert werden.
Von Gitlab vorgegebene Variablen liefern u.A. Informationen über das Projekt, in dem die Pipeline ausgeführt wird.

---

## Variablen - Userdefiniert


```{.yaml data-line-numbers=[6|8]}
variables:
  NAME: "Welt"

greeting_job:
  variables:
    GREETING: "Hallo"
  script: 
    - echo $GREETING, $NAME!
```

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
