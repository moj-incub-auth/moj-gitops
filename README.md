
## Deploy through GitOps

Change into `deployment/overlays/deploy` directory and execute `oc apply -k .` 

This will call on ArgoCD to deploy the following components:

- The ingestion app, used to ingest new or update existing source content
- The search app, performs the search on the content
- The guardrails, for sanitising user input on the search app
- pipelines, tasks and webhooks to automate ingestion of new and updated content as well as application building 


Please read on for additional manual configuration steps required

* Configuring the pipelines
* Setting webhooks on repositories


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

## Setting webhooks on repositories

Webhooks allow for automatically rebuilding of application images and of ingesting source content whenever any updates happen on the underlying repositories thus ensuring that the latest versions are available.

Webhooks require EventListeners to have been defined for every pipeline that is to be called. By the time GitOps (ArgoCD) finished deploying all the required components, EventListeners would have been defined as well. These define services per pipeline and these services need to be exposed as routes. Then the route host needs to be defined in a webhook.

The following commands give an example of the procedure:

```bash
# list the available services
oc get svc -n moj-builder

# select the service to be exposed from the list, e.g. el-dwp-design-listener
# expose the service to create a route
oc expose svc el-dwp-design-listener

# get the hostname of the route
oc get route el-dwp-design-listener -o jsonpath='{.spec.host}'

# in this case: el-dwp-design-listener-moj-builder.apps.cluster-6ldk6.6ldk6.sandbox2187.opentlc.com
#
# from the GitHub web ui, goto Settings -> Webhooks and add a webhook with the following attributes:
# 
# payload = http://<the-host-from-the-previous-step>, i.e.
#           http://el-dwp-design-listener-moj-builder.apps.cluster-6ldk6.6ldk6.sandbox2187.opentlc.com
#
# Content-Type: application/json
#
# Verify that the event type is "Push"
#
# Apply/Add the webhook
```

The above procedure should be done for all repos that webhooks need to be defined.

For bulk definition of webhooks the following script can be used:

**IMPORTANT**
- Use your own GitHub token as the value of `TOKEN`. The value that is in the script is for demonstration purposes only.
- Keep the `LISTENERS` array up-to-date if changes happen in the project

```bash
#!/usr/bin/env bash

# --- Configuration ---
TOKEN="87ds878d989839hhvhvg23hvgh32"     # *** REPLACE THIS WITH YOUR GITHUB TOKEN ***
OWNER="moj-incub-auth"
SECRET="s3cr3t"

declare -A LISTENERS
LISTENERS=(
   ["el-dwp-design-listener"]="design-system"
   ["el-govuk-design-listener"]="govuk-design-system"
   ["el-hmrc-design-listener"]="hmrc-design-system"
   ["el-ingestion-app-listener "]="moj-vector-data"
   ["el-moj-frontend-listener"]="moj-frontend"
   ["el-search-app-listener"]="moj-vector-data"
   ["el-vibe-ui-app-listener"]="vibe-ui"
   ["el-ingestion-ai-app-listener"]="moj-vector-data-v2"
   ["el-nhs-design-listener"]="nhsuk-service-manual"
)

# --- Execution ---
for ROUTE in "${!LISTENERS[@]}"; do
  REPO=${LISTENERS[$ROUTE]}

  oc expose svc $ROUTE
  
  echo "--- Processing: $ROUTE -> $REPO ---"

  # 1. OpenShift Route Check
  if ! oc get route "$ROUTE" &> /dev/null; then
    echo "Skipping: Route $ROUTE not found in OpenShift."
    continue
  fi

  # 2. Determine the Target URL
  TARGET_URL=$(oc get route "$ROUTE" -o jsonpath='{.spec.host}')
  FULL_WEBHOOK_URL="http://$TARGET_URL"

  # 3. Check if Webhook already exists in GitHub
  echo "Checking for existing webhooks in $REPO..."
  EXISTING_HOOKS=$(curl -s -H "Authorization: Bearer $TOKEN" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    "https://api.github.com/repos/$OWNER/$REPO/hooks")

  # Check if the URL string exists in the JSON response
  if echo "$EXISTING_HOOKS" | grep -q "$FULL_WEBHOOK_URL"; then
    echo "Skip: Webhook for $FULL_WEBHOOK_URL already exists in $REPO."
    continue
  fi

  # 4. Create Webhook (only if not found above)
  echo "No match found. Creating new webhook..."
  RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
    -X POST \
    -H "Accept: application/vnd.github+json" \
    -H "Authorization: Bearer $TOKEN" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    "https://api.github.com/repos/$OWNER/$REPO/hooks" \
    -d "{
      \"name\": \"web\",
      \"active\": true,
      \"events\": [\"push\"],
      \"config\": {
        \"url\": \"$FULL_WEBHOOK_URL\",
        \"content_type\": \"json\",
        \"secret\": \"$SECRET\"
      }
    }")

  if [ "$RESPONSE" -eq 201 ]; then
    echo "SUCCESS: Webhook created."
  else
    echo "FAILURE: Could not create webhook (Status: $RESPONSE)."
  fi
  
  echo ""
done
```

