### Carboncopy - Node

```{.yaml .code-more data-line-numbers=[|1-4|6-9|11-15|17-33|]}
Test node:
  extends: .test
  variables:
    APP: node

Build node:
  extends: .build
  variables:
    APP: node

Deploy node to development:
  extends: .deploy to development
  needs: ["Build node"]
  variables:
    APP: node

Deploy node to test:
  extends: .deploy to test
  needs: ["Build node"]
  variables:
    APP: node

Deploy node to stage:
  extends: .deploy to stage
  needs: ["Build node"]
  variables:
    APP: node

Deploy node to production:
  extends: .deploy to production
  needs: ["Deploy node to stage"]
  variables:
    APP: node
```
