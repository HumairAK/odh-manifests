# To get multi-user isolation working:

1. Install odh operator via [olm](https://opendatahub.io/docs/getting-started/quick-installation.html).
2. install openshift pipelines operator via olm 1.7+
3. Deploy dsp: 

```bash
oc new-project data-science-pipelines
oc apply -f ${odh_manifests_repo}/data-science-pipelines/base/auth/kfdef.yaml
``` 

Wait for DSP to come up, `oc get pods -n data-science-pipelines`

4. Give your ocp user access to the user's namespace where runs/experiments will be deployed: 

```bash
cd ${odh_manifests_repo}/data-science-pipelines/base/auth/rbac/user-ns
# replace with ocp username accordingly, if ${USER_NS} does not exist create it: oc new-project ${USER_NS}
sed -i "s/REPLACE-USER/${OCP_USERNAME}/g" rolebinding.yaml 
kustomize build . | oc -n ${USER_NS} apply -f -
```

6. Deploy pipeline-runner sa in ${USER_NS}

```bash
oc -n ${USER_NS} apply -f ${odh_manifests_repo}/data-science-pipelines/base/auth/pipeline-runner-sa.yaml 
```

7. Port-forward api-server pod: 

```bash
oc -n data-science-pipelines port-forward service/ds-pipeline 8888
```

8. Navigate to the UI, retrieve the route from here: 

```bash

echo https://$(oc get routes -n data-science-pipelines ds-pipeline-ui --template={{.spec.host}})
```

Login to the UI, upload a pipelinerun, and try to run it. You can then try to query this run via curl and setting 
the appropriate auth headers:

Retrieve this pipeline run's id: 

```bash
USER_TOKEN=$(oc whoami --show-token)

curl --insecure --location \
--request GET "https://localhost:8888/apis/v1beta1/pipelines" \
--header "Authorization: Bearer ${USER_TOKEN}"
```


8. Navigate to the ml-pipeline route and upload the pipeline provided at `${odh_manifests_repo}/base/auth/flip-coin-pipeline.yaml`

9. Experiments are namespace scoped, and can be created using the following curl: 

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

