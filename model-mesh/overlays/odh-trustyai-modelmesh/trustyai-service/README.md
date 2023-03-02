# TrustyAI-Service

TrustyAI is a service to provide fairness metrics to ModelMesh-served
models.
 
### Installation
To install TrustyAI add the following to the `kfctl` yaml file:

```yaml 
  - kustomizeConfig:
      parameters:
      - name: namespace
        value: opendatahub
      repoRef:
        name: manifests
        path: trustyai-service/base
    name: trustyai-service
```
