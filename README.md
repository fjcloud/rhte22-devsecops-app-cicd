# How to get the Pipelines Deployed

Below steps have been tested with OpenShift Pipelines Operator v1.8.2

## Build Pipeline

1. Create the namespace

    ~~~sh
    oc create namespace rhte22-devsecops-app-ci
    ~~~
2. Create the secret containing the quay credentials

    ~~~sh
    export QUAY_USERNAME=<your_user>
    read -s QUAY_PASSWORD
    export QUAY_PASSWORD
    export QUAY_TOKEN=$(echo -n ${QUAY_USERNAME}:${QUAY_PASSWORD} | base64 -w0)
    envsubst < others/quay-credentials.yaml | oc -n rhte22-devsecops-app-ci create -f -
    ~~~
3. Create the secret required for triggering the pipeline from the webhook

    ~~~sh
    oc -n rhte22-devsecops-app-ci create secret generic webhook-secret --from-literal=secret=v3r1s3cur3
    ~~~
4. Create the required yamls for the pipeline

    ~~~sh
    oc -n rhte22-devsecops-app-ci create -f others/buildah-role.yaml
    oc -n rhte22-devsecops-app-ci create -f tasks/buildah.yaml
    oc -n rhte22-devsecops-app-ci create -f tasks/git-clone.yaml
    oc -n rhte22-devsecops-app-ci create -f tasks/golangci-lint.yaml
    oc -n rhte22-devsecops-app-ci create -f tasks/golang-test.yaml
    oc -n rhte22-devsecops-app-ci create -f tasks/cosign.yaml
    oc -n rhte22-devsecops-app-ci create -f tasks/image-check.yaml
    oc -n rhte22-devsecops-app-ci create -f pipelines/build-pipeline.yaml
    oc -n rhte22-devsecops-app-ci create -f pipelines/build-pipeline-signed.yaml
    oc -n rhte22-devsecops-app-ci create -f pipelines/trigger-template.yaml
    ~~~
5. Generate Cosign KeyPair to sign the container images

    ~~~sh
    cd others/
    cosign generate-key-pair k8s://rhte22-devsecops-app-ci/cosign
    ~~~

6. Go to the [App git repo](https://github.com/ocp-tigers/rhte22-devsecops-app) and configure the webhook as follows

    1. Click on `Settings` -> `Webhooks`
    2. Create the following `Hook`
       1. `Payload URL`: Output of command `oc -n rhte22-devsecops-app-ci get route el-build-pipeline -o jsonpath='http://{.spec.host}'`
       2. `Content type`: application/json
       3. `Secret`: v3r1s3cur3
       4. `Events`: Check **Push Events**, leave others blank
       5. `Active`: Check it
       6. Click on `Add webhook`
