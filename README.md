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
6. Install Stackrox / RHACS using GitOps

    ~~~sh
    oc create -f argoapp-acs-install.yaml
    ~~~
7.  After the Argo App is fully synched and finished properly, check the Stackrox / ACS route:

    ~~~sh
    ACS_ROUTE=$(k get route -n stackrox central -o jsonpath='{.spec.host}')
    curl -Ik https://${ACS_ROUTE}
    ~~~

8. Generate an API Token within RHACS, go to Platform Configuration -> Integrations -> Authentication Tokens -> API Token and generate new API Token.
9. Grab the token generated, and export into the ROX_API_TOKEN variable:

    ~~~sh
    export ROX_API_TOKEN_RAW=<your token>
    ~~~
10. To be able to authenticate from the Tekton Pipelines towards the Stackrox / ACS API, the roxctl tasks used in the pipelines needs to have both ROX_API_TOKEN (generated in one step before) and the ACS Route as well:

    ~~~sh
    read -s ROX_API_TOKEN_RAW
    export ROX_API_TOKEN=$(echo -n $ROX_API_TOKEN_RAW | tr -d '\n' | base64 -w0)
    export ACS_ROUTE=$(echo -n $ACS_ROUTE:443 | tr -d '\n' | base64 -w0)
    envsubst < others/rox-credentials.yaml | oc -n rhte22-devsecops-app-ci create -f -
    ~~~

11. Add the Cosign signature into the Stackrox / ACS integrations. Go to Integrations, Signature, New Integration and add the following:

    ```md
    Integration Name - Cosign-signature
    Cosign Public Name - cosign-pubkey
    Cosign Key Value - Content of cosign.pub generated before
    ```

12. Use the Generic Docker Integration integration to add Quay registry credentials into Stackrox / ACS

13. Create the ACS Policy for the image verification importing the Trusted Signature Image Policy json into the ACS console. Go to Platform Configuration -> Policy Management -> Import Policy.

14. Copy and paste the content of the [ACS Policy pre-generated](https://raw.githubusercontent.com/rcarrata/ocp4-network-security/sign-acs/sign-images/policies/signed-image-policy.json) (or upload the json file).

15. After imported, check the policy generated and select the response method as Inform and enforce. In the policy scope restrict the Policy Scope of the Policy to the specific cluster and namespace (in our case rhte22-devsecops-app-ci) and save the policy.

8. Go to the [App git repo](https://github.com/ocp-tigers/rhte22-devsecops-app) and configure the webhook as follows

    1. Click on `Settings` -> `Webhooks`
    2. Create the following `Hook`
       1. `Payload URL`: Output of command `oc -n rhte22-devsecops-app-ci get route el-build-pipeline -o jsonpath='http://{.spec.host}'`
       2. `Content type`: application/json
       3. `Secret`: v3r1s3cur3
       4. `Events`: Check **Push Events**, leave others blank
       5. `Active`: Check it
       6. Click on `Add webhook`
