---
title: Gitlab CI/CD
author: Tibor Pilz
date: 01.03.2021
---

# Jobs

## Script
## Image

# Pipelines

## Stages

# Variables

## User Defined
## Presets
## Priorität
## Anwendung

# Extends

## Hidden Jobs

# Include

# Rules

## if
## changes
## when

# DAGs

%% ## Pipelines, Stages & Jobs

%% - Definiert in `.gitlab-ci.yml`
%% - Eine Pipeline besteht aus einem oder mehreren Jobs
%% - Jobs werden in Stages eingeteilt
%% - Stages bestimmen die Reihenfolge der Jobs

%% ## Pipelines, Stages & Jobs

%% ```{.yaml data-line-numbers=[1-18|1-3|5-18|5-8|2,6|10-13|15-18|1-18]}
%% stages:
%%   - test
%%   - build

%% test_unit:
%%   stage: test
%%   script:
%%     - echo "running unit tests"

%% test_integration:
%%   stage: test
%%   script:
%%     - echo "running integration tests"

%% build:
%%   stage: build
%%   script:
%%     - echo "building the thing"
%% ```
