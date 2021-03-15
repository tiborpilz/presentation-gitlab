### Carboncopy

```{.yaml .code-more data-line-numbers=[|1-7|9-16|18-27|31-46|31-35,45,46|36-44|48-83|85-115|117-127|129-134|136-141|143-157|159-173|175-205|207-211|208]}
workflow:
  rules:
    # skip all merge rquests labeled as work in progress
    - if: $CI_COMMIT_MESSAGE =~ /-wip$/
      when: never
    # run for all push events and for manual triggers via gitlab web
    - if: '$CI_PIPELINE_SOURCE == "push" || $CI_PIPELINE_SOURCE == "web"'

variables:
  # git applies filters such as git-crypt upon fetch which fails,
  # git clone dosn't, so this resolves the issue
  GIT_STRATEGY: clone
  # this is defines the google chat room where notifications are sent to and includes the credentials
  CHAT_WEBHOOK: ""
  # this is the utility image and needs to manually be version controlled if the content changes
  UTIL_IMAGE: "$CI_REGISTRY_IMAGE/ci-utils:v6"

stages:
  - notification_start
  - util
  - test
  - build
  - deploy to development
  - notification_end
  - deploy to test
  - deploy to stage
  - deploy to production

#### Setup Stage Jobs ####

Notification Start:
  stage: notification_start
  inherit:
    variables: true
  script:
    - wget -qO-
      "$CHAT_WEBHOOK&threadKey=$CI_PIPELINE_ID"
      --header='Content-Type:application/json'
      --post-data="{cards:[{
        'header':{'title':'$CI_COMMIT_REF_NAME','subtitle':'$CI_PIPELINE_SOURCE from $GITLAB_USER_NAME'},
        'sections':[{'widgets':[{'buttons':[{'textButton':{'text':'Pipeline $CI_PIPELINE_ID','onClick':{'openLink':{'url':'$CI_PIPELINE_URL'}}}}]}]}],
        'sections':[{'widgets':[{'textParagraph':{'text':'$CI_COMMIT_MESSAGE'}}]}],
      }]}"
      > /dev/null
  rules:
    - if: $CHAT_WEBHOOK

Build Utils Image:
  stage: util
  image: docker:dind
  inherit:
    variables: true
  before_script:
    - test -z "$CHAT_WEBHOOK" || wget -qO-
      "$CHAT_WEBHOOK&threadKey=$CI_PIPELINE_ID"
      --header='Content-Type:application/json'
      --post-data="{cards:[{
        'header':{'title':'$CI_JOB_STAGE','subtitle':'$CI_JOB_NAME'},
        'sections':[{'widgets':[{'buttons':[{'textButton':{'text':'Job $CI_JOB_ID','onClick':{'openLink':{'url':'$CI_JOB_URL'}}}}]}]}],
        'sections':[{'widgets':[{'textParagraph':{'text':'started'}}]}],
      }]}"
      > /dev/null
  script:
    - printf '%s' "$CI_REGISTRY_PASSWORD" | docker login --username $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
    - DOCKER_CLI_EXPERIMENTAL=enabled docker manifest inspect $UTIL_IMAGE >/dev/null
      || {
        docker build --network host -t $UTIL_IMAGE ops/ci-utils;
        docker push $UTIL_IMAGE;
      }
  after_script:
    - test -z "$CHAT_WEBHOOK" || wget -qO-
      "$CHAT_WEBHOOK&threadKey=$CI_PIPELINE_ID"
      --header='Content-Type:application/json'
      --post-data="{cards:[{
        'header':{'title':'$CI_JOB_STAGE','subtitle':'$CI_JOB_NAME'},
        'sections':[{'widgets':[{'buttons':[{'textButton':{'text':'Job $CI_JOB_ID','onClick':{'openLink':{'url':'$CI_JOB_URL'}}}}]}]}],
        'sections':[{'widgets':[{'textParagraph':{'text':'finished'}}]}],
      }]}"
      > /dev/null
  rules:
    - changes:
      - ops/ci-utils/*
      - .gitlab-ci.yml

.util:
  image: $UTIL_IMAGE
  inherit:
    variables: true
  before_script:
    - test -z "${CHAT_WEBHOOK-}" || wget -qO-
      "$CHAT_WEBHOOK&threadKey=$CI_PIPELINE_ID"
      --header='Content-Type:application/json'
      --post-data="{cards:[{
        'header':{'title':'$CI_JOB_STAGE','subtitle':'$CI_JOB_NAME'},
        'sections':[{'widgets':[{'buttons':[{'textButton':{'text':'Job $CI_JOB_ID','onClick':{'openLink':{'url':'$CI_JOB_URL'}}}}]}]}],
        'sections':[{'widgets':[{'textParagraph':{'text':'started'}}]}],
      }]}"
      > /dev/null
    - printf '%s' "$CI_REGISTRY_PASSWORD" | docker login --username $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
    - test $CI_PROJECT_PATH = "hosting/carboncopy" || {
        kubectl config set-cluster avenga-public --server=${CLUSTER_ADDRESS:?};
        kubectl config set-credentials avenga-public --username=${RANCHER_USERNAME:?} --password=${RANCHER_PASSWORD:?};
        kubectl config set-context avenga-public --cluster=avenga-public --user=avenga-public;
      }
    - test -z "$GIT_CRYPT_KEY" || echo "$GIT_CRYPT_KEY" | base64 -d | git crypt unlock -
  after_script:
    - test -z "$CHAT_WEBHOOK" || wget -qO-
      "$CHAT_WEBHOOK&threadKey=$CI_PIPELINE_ID"
      --header='Content-Type:application/json'
      --post-data="{cards:[{
        'header':{'title':'$CI_JOB_STAGE','subtitle':'$CI_JOB_NAME'},
        'sections':[{'widgets':[{'buttons':[{'textButton':{'text':'Job $CI_JOB_ID','onClick':{'openLink':{'url':'$CI_JOB_URL'}}}}]}]}],
        'sections':[{'widgets':[{'textParagraph':{'text':'finished'}}]}],
      }]}"
      > /dev/null

.test:
  stage: test
  extends: .util
  script:
    - ops/test
  rules:
    # always allow manual tests
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
      when: manual
    # automatically run for any nondefault branch
    - if: '$CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH'

.build:
  stage: build
  extends: .util
  script:
    - ops/build
    - ops/push

.deployment:
  extends: .util
  script:
    - ops/regcred
    - ops/deploy
    - ops/wait

Notification End Success:
  stage: notification_end
  script:
    - test -z "$CHAT_WEBHOOK" || wget -qO-
      "$CHAT_WEBHOOK&threadKey=$CI_PIPELINE_ID"
      --header='Content-Type:application/json'
      --post-data="{cards:[{
        'header':{'title':'$CI_COMMIT_REF_NAME','subtitle':'$CI_PIPELINE_SOURCE from $GITLAB_USER_NAME'},
        'sections':[{'widgets':[{'buttons':[{'textButton':{'text':'Pipeline $CI_PIPELINE_ID','onClick':{'openLink':{'url':'$CI_PIPELINE_URL'}}}}]}]}],
        'sections':[{'widgets':[{'textParagraph':{'text':'Automatic Pipeline Successfull, ready for manual deployment.'}}]}],
      }]}"
      > /dev/null
  rules:
    - if: $CHAT_WEBHOOK
      when: on_success

Notification End Failure:
  stage: notification_end
  script:
    - test -z "$CHAT_WEBHOOK" || wget -qO-
      "$CHAT_WEBHOOK&threadKey=$CI_PIPELINE_ID"
      --header='Content-Type:application/json'
      --post-data="{cards:[{
        'header':{'title':'$CI_COMMIT_REF_NAME','subtitle':'$CI_PIPELINE_SOURCE from $GITLAB_USER_NAME'},
        'sections':[{'widgets':[{'buttons':[{'textButton':{'text':'Pipeline $CI_PIPELINE_ID','onClick':{'openLink':{'url':'$CI_PIPELINE_URL'}}}}]}]}],
        'sections':[{'widgets':[{'textParagraph':{'text':'Automatic Pipeline Failure, deployment blocked.'}}]}],
      }]}"
      > /dev/null
  rules:
    - if: $CHAT_WEBHOOK
      when: on_failure

.deploy to development:
  stage: deploy to development
  extends: .deployment
  variables:
    ENVIRONMENT: development
  rules:
    # always allow manual deployment to development
    - when: manual
    # automatically run for the default branch
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

.deploy to test:
  stage: deploy to test
  extends: .deployment
  variables:
    ENVIRONMENT: test
  when: manual

.deploy to stage:
  stage: deploy to stage
  extends: .deployment
  variables:
    ENVIRONMENT: stage
  when: manual

.deploy to production:
  stage: deploy to production
  extends: .deployment
  variables:
    ENVIRONMENT: production
  when: manual

include:
  - local: '/apps/nginx/.gitlab-ci.yml'
  - local: '/apps/node/.gitlab-ci.yml'
  - local: '/apps/wordpress/.gitlab-ci.yml'
```
---
