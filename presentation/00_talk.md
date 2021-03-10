---
title: Gitlab CI/CD
author: Tibor Pilz
date: 15.03.2021
---

## Pipelines, Stages & Jobs

- Gitlab CI/CD führt Aktionen aus
- Konfiguriert durch `gitlab-ci.yml`
- Konfiguration beschreibt eine Pipeline
- Eine Pipeline besteht aus einem oder mehreren Jobs
- Jobs werden in Stages eingeteilt
- Stages bestimmen die Reihenfolge der Jobs

---

## Jobs

```{.yaml data-line-numbers=[1-2|1|2|1-2]}
hello_world_job:
  script: echo "Hallo, Welt!"
```

::: notes

- Ein Job ist die kleinste Einheit einer Pipeline
- Job ist ein Top-Level Objekt
- Bis auf Ausnahmen wird der Key frei gewählt und ist der Name des Jobs
- Die Ausnahmen sind Gitlab Keywords
- Script ist notwendig, beschreibt die Aktion, die ausgeführt werden soll

:::

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

- Um die Reihenfolge der Jobs festzulegen, sind Stages notwendig
- Stages werden als Top-Level Array definiert
- (`stages` ist ein Beispiel für einen ungültigen Job-Namen)

:::

---

## Job - Script

::: notes

Eine oder mehrere Zeilen Shellscript.
Der Exit-Code des Shellscripts bestimmt, ob der Job fehlschlägt oder erfolgreich ist.
Sofern nicht anders konfiguriert, stoppen fehlgeschlagene Scripts die Pipeline und lassen sie fehlschlagen.

:::

---

## Job - Image

Jobs werden per Container ausgeführt.
Das Image kann pro Pipeline oder pro Job angegeben werden.

---

## Job - Image - Example

```{.yaml data-line-numbers=[1-5|2|4-5]}
node_job:
  image: node:12
  script:
    - npm install
    - npm run test
```

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
