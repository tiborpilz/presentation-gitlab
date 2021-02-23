# Anatomie einer Gitlab Pipeline

```yaml [1]
test:
  image: node:12
  script:
    - npm run install
    - npm run test
```
