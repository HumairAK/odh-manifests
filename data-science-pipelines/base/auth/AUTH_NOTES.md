# To get multi-user isolation working:

1. Install rhods/odh operator
2. `oc apply -f ${odh-manifest-repo}/base/auth/kfdef.yaml`
3. scale down odh operator `oc patch deployment rhods-operator -n redhat-ods-operator --type=merge -p '{"spec":{"replicas": 0}}'`
4. Add minio port to `NetworkPolicy` named `redhat-ods-applications`: 

```yaml
spec:
  podSelector: {}
  ingress:
    - ports:
        ...
        - protocol: TCP
          port: 9000
```

And also ensure any namespaces that you want to kfp to deploy to have the label: `opendatahub.io/generated-namespace: 'true'`
So that pipelines in that namespace can access minio in `redhat-ods-applications`. 

3. For this poc we give all authenticated users access to `get` `namespaces` to get past api-server's oauth-proxy.
   To do this deploy the rbac in:

```bash
cd ${odh-manifest-repo}/base/auth/rbac/authenticated-access
kustomize build . | oc apply -f -
```

4. Give your ocp user access to the user's namespace where runs/experiments will be deployed: 

```bash
cd ${odh-manifest-repo}/base/auth/rbac/user-ns
# replace with ocp username accordingly
yq e -i '.subjects[0].name = "REPLACE-USER"' rolebinding.yaml 
kustomize build . | oc -n ${USER-NS} oc apply -f -
```

5. Deploy pipeline-runner sa in user-ns `oc -n ${USER-NS} apply -f ${odh-manifest-repo}/base/auth/pipeline-runner-sa.yaml`

Retrieve this pipeline run's id: 

```bash
USER_TOKEN=$(oc whoami --show-token)

curl --insecure --location \
--request GET "https://localhost:8888/apis/v1beta1/pipelines" \
--header "Authorization: Bearer ${USER_TOKEN}"
```

6. Port-forward api-server pod: `oc -n redhat-ods-applications port-forward service/ds-pipeline 8888`

7. Navigate to the ml-pipeline route and upload the pipeline provided at `${odh-manifest-repo}/base/auth/flip-coin-pipeline.yaml`

8. Experiments are namespace scoped, and can be created using the following curl: 

```bash

# replace EXPERIMENT_DESCRIPTION, EXPERIMENT_NAME, USER_NS accordingly
curl --insecure --location --request POST 'https://localhost:8888/apis/v1beta1/experiments' \
--header "Authorization: Bearer ${USER_TOKEN}" \
-H "Content-Type: application/json" \
--data-raw '{
    "description": "some-description",
    "name": "exp-1",
    "resource_references": [
        {
            "key": {
                "id": "hukhan",
                "type": "NAMESPACE"
            },
            "relationship": "OWNER"
        }
    ]
}'
```

List the experiment: 
```bash
curl --insecure --location \
--request GET "https://localhost:8888/apis/v1beta1/experiments?resource_reference_key.id=${USER_NS}&resource_reference_key.type=NAMESPACE" \
--header "Authorization: Bearer ${USER_TOKEN}"
```

Note that this experiment belongs to ${USER_NS}, now if we create a run in this experiment then it will be deployed in 
this experiments namespace: 

```bash
curl --insecure --location --request POST 'https://localhost:8888/apis/v1beta1/runs' \
--header "Authorization: Bearer ${USER_TOKEN}" \
-H "Content-Type: application/json" \
--data-raw '{
    "description": "",
    "name": "some-run-name",
    "pipeline_spec": {
        "parameters": []
    },
    "resource_references": [
        {
            "key": {
                "id": "$YOUR_EXPERIMENTS_ID",
                "type": "EXPERIMENT"
            },
            "relationship": "OWNER"
        },
        {
            "key": {
                "id": "$YOUR_PIPELINE_ID",
                "type": "PIPELINE_VERSION"
            },
            "relationship": "CREATOR"
        }
    ],
    "service_account": ""
}'

```

List runs in this namespace to see if it was created: 

```bash
curl --insecure  \
--location \
--request GET "https://localhost:8888/apis/v1beta1/runs?resource_reference_key.id=${USER_NS}&resource_reference_key.type=NAMESPACE" \
--header "Authorization: Bearer ${USER_TOKEN}"
```

Confirm pipeline pods are running in this namespace: 

```bash
oc get pods -n ${USER_NS}
oc get pipelineruns -n ${USER_NS}
```

