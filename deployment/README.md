
## Deploy through GitOps

Change into `deployment/overlays/deploy` directory and execute `oc apply -k .` 

This will call on ArgoCD to deploy the following components:

- The ingestion app, used to ingest new or update existing source content
- The search app, performs the search on the content
- The guardrails, for sanitising user input on the search app
- pipelines, tasks and webhooks to automate ingestion of new and updated content as well as application building 


## Additional configuration needed for the pipelines

The pipelines are installed in a different namespace than the rest of the components and need to pull images from the Red Hat Image Registry. In order to be able to deposit images to different namespaces as well as accessing the Red Hat Image Registry additional privileges need to be bestowed upon them through the following steps.

### Create a Red Hat Registry Service Account

Visit [Red Hat Customer Portal - Token Generation](https://access.redhat.com/terms-based-registry/accounts) and create a new token. If you have already created one, use the search option to locate it. Take a note of the `username` and of the `token` and create the `redhat-registry-credentials` secret by replacing the values in the following manifest:

Please note:

> `USERNAME` usually looks like `7809811|token-name` 

> `TOKEN` is a long continuous string of characters, like `eyJhbGciOiJSUzUxMiJ9.eyJzdWIiOiIzNzA4YTFjOGM0Nzg0NTEyYTI1MzVjZTY3NjYxNmVjYyJ9.D4.....`. Make sure you copy all of them. The `TOKEN` SHOULD NOT have any new lines or spaces.


```bash
oc create secret docker-registry redhat-registry-credentials \
    --docker-server=registry.redhat.io \
    --docker-username='<USERNAME>' \
    --docker-password='<TOKEN>' \
    --docker-email=<YOUR_E-MAIL>
```

Finally, link the secret to the `pipeline` service account through the following

```bash
oc secrets link pipeline redhat-registry-credentials --for=pull
```

### Grant permission to deposit image to a different namespace

Through the following grant the permission to the `pipeline` service account in the `moj-builder` namespace to deposit images in the `moj-vector-data` namespace where the applications are deployed is granted.

```bash
oc policy add-role-to-user system:image-builder system:serviceaccount:$(oc project -q):pipeline -n moj-vector-data
```

### Create a long lived token for the pipeline

Latest version of Openshift create short-lived tokens that when expire would prevent the pipelines from running. To counter that a long lived token needs to be created and bound to the `pipeline` service account by applying the following manifest:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pipeline-long-lived-token
  annotations:
    kubernetes.io/service-account.name: pipeline
type: kubernetes.io/service-account-token
```

### Merge the two credentials together

Using the following commands the two credentials created above are merged into a single one named `pipeline-pull-push-auth` which is used by the pipelines. Using this merged secret allows the pipelines to both pull images from the Red Hat Image Registry and push to other namespaces (here `moj-vector-data`).

```bash
# Get the raw token string
export SA_TOKEN=$(oc get secret pipeline-long-lived-token -o jsonpath='{.data.token}' | base64 -d)

# Create the base64 auth string (serviceaccount:TOKEN)
export AUTH_STRING=$(echo -n "serviceaccount:${SA_TOKEN}" | base64 -w 0)

# Get the Red Hat auth block first
oc get secret redhat-registry-credentials -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > rh_config.json

# Use jq to inject the internal registry auth into the Red Hat config
jq --arg auth "$AUTH_STRING" '.auths["image-registry.openshift-image-registry.svc:5000"] = {"auth": $auth}' rh_config.json > final-config.json

oc delete secret pipeline-pull-push-auth --ignore-not-found

oc create secret generic pipeline-pull-push-auth \
  --from-file=.dockerconfigjson=final-config.json \
  --type=kubernetes.io/dockerconfigjson

```

