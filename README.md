# pipeline

The pipeline tutorial walks you through creating an end to end deployment pipeline for Kubernetes using [Cloud Container Builder](https://cloud.google.com/container-builder/).

## The Application

This tutorial will set up a pipeline to deploy the [pipeline application](https://github.com/kelseyhightower/pipeline-application), a simple Go application with the following HTTP endpoints:

 * `/` - responds with "Hello world!"
 * `/health` - responds with HTTP status code 200
 * `/version` - responds with the application version (v2.0.0)

## The Deployment Pipeline

This tutorial will leverage [Cloud Container Builder](https://cloud.google.com/container-builder/) to set up the following pipeline:

 * changes pushed to any branch, except the master branch, on the [pipeline application](https://github.com/kelseyhightower/pipeline-application) repo will trigger a container build and a git commit to the [pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging) repo, which updates the pipeline deployment Kubernetes configuration file with the new container image.
 * changes pushed to the [pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging) repo will trigger a rolling deployment to a Kubernetes staging cluster.
 * tags on the [pipeline application](https://github.com/kelseyhightower/pipeline-application) repo will trigger a container build based on the tag name and a git commit to the [pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa) repo, which updates the pipeline deployment Kubernetes configuration file with the new container image.
 * changes pushed to the [pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa) repo will trigger a rolling deployment to a Kubernetes QA cluster.
 * successful deployments to QA will trigger a pull request on the [pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production) repo.
 * changes pushed to the master branch on the [pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production) repo will trigger a rolling deployment to a Kubernetes production cluster.

 > The rolling update to the production cluster is gated by a pull request to the pipeline-infrastructure-production repo.

## Tutorial

### Services

The following services are required to complete this tutorial:

* [GitHub](https://github.com)
* [Google Cloud Platform](https://console.cloud.google.com/freetrial)

### Enable Google Cloud Platform APIs

In this section you will enable the Google Cloud Platform APIs required to complete this tutorial.

Enable the required GCP APIs:

```
gcloud services enable --async \
  container.googleapis.com \
  cloudapis.googleapis.com \
  cloudbuild.googleapis.com \
  sourcerepo.googleapis.com \
  compute.googleapis.com \
  storage-component.googleapis.com \
  containerregistry.googleapis.com \
  cloudkms.googleapis.com \
  logging.googleapis.com \
  cloudfunctions.googleapis.com
```

In may take several minutes before the GCP APIs become enabled and ready for use. In the meanwhile it's safe to continue with the tutorial. At any point you can use the `gcloud` command to list enabled services:

```
gcloud services list --enabled
```

> output

```
NAME                               TITLE
bigquery-json.googleapis.com       BigQuery API
clouddebugger.googleapis.com       Stackdriver Debugger API
datastore.googleapis.com           Google Cloud Datastore API
source.googleapis.com              Legacy Cloud Source Repositories API
storage-component.googleapis.com   Google Cloud Storage
pubsub.googleapis.com              Google Cloud Pub/Sub API
container.googleapis.com           Google Container Engine API
storage-api.googleapis.com         Google Cloud Storage JSON API
logging.googleapis.com             Stackdriver Logging API
resourceviews.googleapis.com       Google Compute Engine Instance Groups API
replicapool.googleapis.com         Google Compute Engine Instance Group Manager API
cloudapis.googleapis.com           Google Cloud APIs
sourcerepo.googleapis.com          Cloud Source Repositories API
deploymentmanager.googleapis.com   Google Cloud Deployment Manager V2 API
containerregistry.googleapis.com   Google Container Registry API
monitoring.googleapis.com          Stackdriver Monitoring API
compute.googleapis.com             Google Compute Engine API
sql-component.googleapis.com       Google Cloud SQL
cloudkms.googleapis.com            Google Cloud Key Management Service (KMS) API
cloudtrace.googleapis.com          Stackdriver Trace API
servicemanagement.googleapis.com   Google Service Management API
replicapoolupdater.googleapis.com  Google Compute Engine Instance Group Updater API
cloudbuild.googleapis.com          Google Cloud Container Builder API
cloudfunctions.googleapis.com      Google Cloud Functions API
```

### Client Tools

The following client tools are required to complete this tutorial:

 * [hub](https://github.com/github/hub) 2.3.0+
 * [hub-credential-helper](https://github.com/kelseyhightower/hub-credential-helper) 0.0.1+
 * [git](https://git-scm.com/downloads) 2.14.0+
 * [gcloud](https://cloud.google.com/sdk) 179.0.0+
 * [kubectl](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.8.md#downloads-for-v183) 1.8.0+

### Create Kubernetes Clusters

In this section you will create three Kubernetes clusters using [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine). Each cluster will represent one of the following environments: staging, qa, production.

Create the `staging`, `qa`, and `production` Kubernetes clusters:

```
for e in staging qa production; do
  gcloud container clusters create ${e} --async --num-nodes 1
done
```

It may take up to 5 minutes

At this point you should have three Kubernetes clusters running:

```
gcloud container clusters list
```
```
NAME        ZONE        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
production  us-west1-c  1.7.8-gke.0     XX.XXX.XXX.XX  n1-standard-1  1.7.8-gke.0   1          RUNNING
qa          us-west1-c  1.7.8-gke.0     XX.XXX.XX.XX   n1-standard-1  1.7.8-gke.0   1          RUNNING
staging     us-west1-c  1.7.8-gke.0     XX.XXX.XXX.XX  n1-standard-1  1.7.8-gke.0   1          RUNNING
```

### Generate a GitHub API Token

In this section you will generate a [GitHub API Token](https://github.com/blog/1509-personal-api-tokens) which will be used to automate the GitHub related tasks throughout this tutorial.

Generate a GitHub token using the official [guide](https://github.com/blog/1509-personal-api-tokens). While creating the token, set the token description to "pipeline", and check the `repo` and `admin:repo_hook` scopes.

![Image of GitHub UI](images/create-github-token.png)

Save the token in the `GITHUB_TOKEN` environment variable:

```
export GITHUB_TOKEN="<token>"
```

Your GitHub username will be used to automate GitHub related tasks including forking the GitHub repositories necessary to complete this tutorial and creating [GitHub webhooks](https://developer.github.com/webhooks/). Save your GitHub username in the `GITHUB_USERNAME` env var:

```
export GITHUB_USERNAME="<github-username>"
```

### Create a Hub Configuration File

In this section you will generate a [hub configuration file](https://hub.github.com/hub.1.html#CONFIGURATION) to hold the GitHub credentials generated in the previous section. The hub configuration file is used by the `hub` command line utility when automating GitHub related tasks.

Create a hub configuration file in the current directory:

```
cat <<EOF > hub
github.com:
  - protocol: https
    user: ${GITHUB_USERNAME}
    oauth_token: ${GITHUB_TOKEN}
EOF
```

Set the `HUB_CONFIG` environment variable to point to the hub configuration file:

```
HUB_CONFIG="${PWD}/hub"
```

### Encrypt the Hub Configuration File and upload to Google Cloud Storage

In this section you will encrypt the hub configuration file using the [Google Key Management Service](https://cloud.google.com/kms) (KMS) and upload the encrypted file to a [Google Cloud Storage](https://cloud.google.com/storage) (GCS) bucket, which will make the hub configuration file securely available during any automated build steps performed by Cloud Container Builder in the future.

#### Create a KMS Keyring and Encryption Key

A KMS keyring and encryption key is required to encrypt the hub configuration file.

Create a new keyring:

```
gcloud kms keyrings create pipeline --location=global
```

Generate a new encryption key:

```
gcloud kms keys create github \
  --location=global \
  --keyring=pipeline \
  --purpose=encryption
```

Encrypt the hub configuration file using the `pipeline` keyring and the `github` encryption key:

```
gcloud kms encrypt \
  --plaintext-file "${HUB_CONFIG}" \
  --ciphertext-file hub.enc \
  --location=global \
  --keyring=pipeline \
  --key=github
```

#### Upload the Encrypted Hub Configuration File to GCS

In this section you will create a GCS bucket and upload the encrypted hub configuration file to it.

Retrieve the active GCP project ID and store it in the `PROJECT_ID` env var:

```
PROJECT_ID=$(gcloud config get-value core/project)
```

Create a GCS bucket that will hold the encrypted hub configuration file:

```
gsutil mb gs://${PROJECT_ID}-pipeline-configs
```

Upload the encrypted hub configuration file to the pipeline configs GCS bucket:

```
gsutil cp hub.enc gs://${PROJECT_ID}-pipeline-configs/
```

#### Grant the Container Builder Service Account Access to the GitHub Encryption Key

In this section you will grant access to the `github` encrypt key to the Container Builder service account. Performing these steps will enable Container Builder to decrypt the hub configuration file during any automated build.

Retrieve the active GCP project number and store it in the `PROJECT_NUMBER` env var:

```
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')
```

Grant the Container Builder service account access to the `github` encryption key:

```
gcloud kms keys add-iam-policy-binding github \
  --location=global \
  --keyring=pipeline \
  --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role=roles/cloudkms.cryptoKeyEncrypterDecrypter
```

Grant the Container Builder service account access to the Google Container Engine API:

```
gcloud projects add-iam-policy-binding ${PROJECT_NUMBER} \
  --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role=roles/container.developer
```

### Setup the Pipeline Repositories

In this section you will fork the following GitHub repositories to your own GitHub account:

* [kelseyhightower/pipeline-application](https://github.com/kelseyhightower/pipeline-application)
* [kelseyhightower/pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging)
* [kelseyhightower/pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa)
* [kelseyhightower/pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production)

Fork the pipeline application and infrastructure repositories:

```
REPOS=(
  pipeline-application
  pipeline-infrastructure-staging
  pipeline-infrastructure-qa
  pipeline-infrastructure-production
)
```

```
for repo in ${REPOS[@]}; do
  hub clone "https://github.com/kelseyhightower/${repo}.git"
  cd ${repo}/
  hub fork
  cd -
  rm -rf ${repo}
done
```

At this point the pipeline application and infrastructure repositories have been forked to your GitHub account and can used as part of your own deployment pipeline.


### Mirror GitHub Repositories to Cloud Source Repositories

Currently Container Builder only supports build triggers on [Cloud Source Repositories](https://cloud.google.com/source-repositories)(CSR). In this section you will create the Cloud Source Repositories that will mirror each of the GitHub repositories created in the previous section.

```
REPOS=(
  pipeline-application
  pipeline-infrastructure-staging
  pipeline-infrastructure-qa
  pipeline-infrastructure-production
)
```

For each GitHub repository create a Cloud Source Repository to mirror it, then preform the initial synchronization:

```
for r in ${REPOS[@]}; do
  gcloud source repos create ${r}
  git clone --mirror https://github.com/${GITHUB_USERNAME}/${r}
  git --git-dir ${r}.git push --mirror \
    --repo "https://source.developers.google.com/p/${PROJECT_ID}/r/${r}"
done
```

At this point the GitHub repositories are mirrored to your Cloud Source Repositories. To keep the Cloud Source repositories synchronized deploy the [reposync webhook](https://github.com/kelseyhightower/reposync) to your project.

#### Deploy the Repo Sync WebHook

In this section you will use [Google Cloud Functions](https://cloud.google.com/functions/) to host the [reposync](https://github.com/kelseyhightower/reposync) webhook used to keep the Cloud Source Repositories in sync with the corresponding GitHub repositories.

Download the `reposync` cloud function:

```
wget https://github.com/kelseyhightower/reposync/releases/download/0.0.1/reposync-cloud-function-0.0.1.zip
```

```
unzip reposync-cloud-function-0.0.1.zip
```

Create a [Google Cloud Storage](https://cloud.google.com/storage) bucket to host the `reposync` cloud function source tree:

```
gsutil mb gs://${PROJECT_ID}-pipeline-functions
```

Deploy the `reposync` cloud function:

```
gcloud beta functions deploy reposync \
  --source reposync-cloud-function-0.0.1 \
  --entry-point F \
  --stage-bucket ${PROJECT_ID}-pipeline-functions \
  --trigger-http
```

### Create the GitHub Webhooks

In this section you will configure each GitHub repository to send [push events](https://developer.github.com/webhooks/#events) to the `reposync` webhook.

Retrieve the `reposync` webhook URL from the Cloud Functions API:

```
WEBHOOK_URL=$(gcloud beta functions describe reposync \
  --format='value(httpsTrigger.url)')
```

Create a webhook configuration payload as defined in the [GitHub Webhooks API guide](https://developer.github.com/v3/repos/hooks/#create-a-hook):

```
cat <<EOF > github-webhook-config.json
{
  "name": "web",
  "active": true,
  "events": [
    "push"
  ],

  "config": {
    "secret": "pipeline",
    "url": "${WEBHOOK_URL}",
    "content_type": "json"
  }
}
EOF
```

Create a wehbook on each pipeline application and infrastructure GitHub repository using the `github-webhook-config.json` webhook configuration payload created in the previous step:

```
for repo in ${REPOS[@]}; do
  curl -X POST "https://api.github.com/repos/${GITHUB_USERNAME}/${repo}/hooks" \
    -H "Content-Type: application/json" \
    -H "Accept: application/vnd.github.v3+json" \
    -u "${GITHUB_USERNAME}:${GITHUB_TOKEN}" \
    --data-binary @github-webhook-config.json
done
```

Each GitHub repositories is now set to send push events to the `reposync` webhook.

### Create the Cloud Container Builder Build Triggers

In this section you will create the Cloud Container Builder build triggers necessary to establish an end-to-end build pipeline described at the start of this tutorial.

Retrieve the default compute zone and store it in the `COMPUTE_ZONE` env var:

```
export COMPUTE_ZONE=$(gcloud config get-value compute/zone)
```

Create a build trigger that rebuilds the pipeline application container image and updates the Kubernetes deployment configuration files for the staging cluster. This trigger will fire when changes are pushed to the `${GITHUB_USERNAME}/pipeline-application` GitHub repository on any branch except the master.

```
cat <<EOF > pipeline-staging-build-trigger.json
{
  "triggerTemplate": {
    "projectId": "${PROJECT_ID}",
    "repoName": "pipeline-application",
    "branchName": "[^(?!.*master)].*"
  },
  "description": "pipeline-staging-build",
  "substitutions": {
    "_GITHUB_USERNAME": "${GITHUB_USERNAME}",
    "_KMS_KEY": "github",
    "_CLOUDSDK_COMPUTE_ZONE": "${COMPUTE_ZONE}",
    "_CLOUDSDK_CONTAINER_CLUSTER": "staging",
    "_KMS_KEYRING": "pipeline"
  },
  "filename": "staging/cloudbuild.yaml"
}
EOF
```

Create a build trigger that rebuilds the pipeline application container image and updates the Kubernetes deployment configuration files for the qa cluster. This trigger will fire when a new tag is pushed to the `${GITHUB_USERNAME}/pipeline-application` GitHub repository.

```
cat <<EOF > pipeline-qa-build-trigger.json
{
  "triggerTemplate": {
    "projectId": "${PROJECT_ID}",
    "repoName": "pipeline-application",
    "tagName": ".*"
  },
  "description": "pipeline-qa-build",
  "substitutions": {
    "_KMS_KEYRING": "pipeline",
    "_GITHUB_USERNAME": "${GITHUB_USERNAME}",
    "_KMS_KEY": "github",
    "_CLOUDSDK_COMPUTE_ZONE": "${COMPUTE_ZONE}",
    "_CLOUDSDK_CONTAINER_CLUSTER": "qa"
  },
  "filename": "qa/cloudbuild.yaml"
}
EOF
```

Create a build trigger that applies the Kubernetes deployment configuration files for the staging cluster. This trigger will fire when changes are pushed to the `${GITHUB_USERNAME}/pipeline-infrastructure-staging` GitHub repository on the master branch.

```
cat <<EOF > pipeline-staging-deployment-trigger.json
{
  "triggerTemplate": {
    "projectId": "${PROJECT_ID}",
    "repoName": "pipeline-infrastructure-staging",
    "branchName": "master"
  },
  "description": "pipeline-staging-deployment",
  "substitutions": {
    "_CLOUDSDK_CONTAINER_CLUSTER": "staging",
    "_CLOUDSDK_COMPUTE_ZONE": "${COMPUTE_ZONE}"
  },
  "filename": "cloudbuild.yaml"
}
EOF
```

Create a build trigger that applies the Kubernetes deployment configuration files for the qa cluster. This trigger will fire when changes are pushed to the `${GITHUB_USERNAME}/pipeline-infrastructure-qa` GitHub repository on the master branch.

```
cat <<EOF > pipeline-qa-deployment-trigger.json
{
  "triggerTemplate": {
    "projectId": "${PROJECT_ID}",
    "repoName": "pipeline-infrastructure-qa",
    "branchName": "master"
  },
  "description": "pipeline-qa-deployment",
  "substitutions": {
    "_KMS_KEYRING": "pipeline",
    "_GITHUB_USERNAME": "${GITHUB_USERNAME}",
    "_CLOUDSDK_COMPUTE_ZONE": "${COMPUTE_ZONE}",
    "_CLOUDSDK_CONTAINER_CLUSTER": "qa",
    "_KMS_KEY": "github"
  },
  "filename": "cloudbuild.yaml"
}
EOF
```

Create a build trigger that applies the Kubernetes deployment configuration files for the production cluster. This trigger will fire when changes are pushed to the `${GITHUB_USERNAME}/pipeline-infrastructure-production` GitHub repository on the master branch.

```
cat <<EOF > pipeline-production-deployment-trigger.json
{
  "triggerTemplate": {
    "projectId": "${PROJECT_ID}",
    "repoName": "pipeline-infrastructure-production",
    "branchName": "master"
  },
  "description": "pipeline-production-deployment",
  "substitutions": {
    "_CLOUDSDK_COMPUTE_ZONE": "${COMPUTE_ZONE}",
    "_CLOUDSDK_CONTAINER_CLUSTER": "production"
  },
  "filename": "cloudbuild.yaml"
}
EOF
```

Create a cloud build trigger for each build trigger configuration file:

```
BUILD_TRIGGER_CONFIGS=(
  pipeline-staging-build-trigger.json
  pipeline-qa-build-trigger.json
  pipeline-staging-deployment-trigger.json
  pipeline-qa-deployment-trigger.json
  pipeline-production-deployment-trigger.json
)
```

```
for config in ${BUILD_TRIGGER_CONFIGS[@]}; do
  curl -X POST \
    https://cloudbuild.googleapis.com/v1/projects/${PROJECT_ID}/triggers \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
    --data-binary @${config}
done
```

At this point all the build triggers are in place. It's now time to test the build pipeline.

### Test the Pipeline

In this section you will test the Cloud Builder build pipeline by making modifications to the pipeline application and observing how the each change propagates through the staging, qa, and production environments.

Ensure you have the cluster credentials for each Kubernetes cluster:

```
for e in staging qa production; do
  gcloud container clusters get-credentials ${e}
done
```

Verify no pods are currently running in any environment:

```
for e in staging qa production; do
  kubectl get pods --context "gke_${PROJECT_ID}_${COMPUTE_ZONE}_${e}"
done
```

#### Modify the Pipeline Application

In this section you will modify the pipeline application and push the changes to a new branch on your pipeline-application GitHub repository.

Configure git to ensure the hub command line utility uses the HTTPS protocol will working with GitHub repositories:

```
git config --global hub.protocol https
```

Configure a git credential helper to use the `hub-credential-helper` utility when authenticating to GitHub:

```
git config --global credential.https://github.com.helper /usr/local/bin/hub-credential-helper
```

Clone the `pipeline-application` GitHub repository to the current directory:

```
hub clone ${GITHUB_USERNAME}/pipeline-application
```

Change into the `pipeline-application` directory and create a new branch named `new-message`:

```
cd pipeline-application
```

```
git checkout -b new-message
```

Modify the message return for HTTP requests to the pipeline application:

```
sed "s/world/${GITHUB_USERNAME}/g" main.go > main.go.new
```

```
mv main.go.new main.go
```

> The syntax for in-place sed updates does not work consistently across operating systems so we are forced to create a temporary file and use it to overwrite the target of our changes.

Review the changes to the pipeline application:

```
git diff
```
```
diff --git a/main.go b/main.go
index 2f76589..0b08a59 100644
--- a/main.go
+++ b/main.go
@@ -21,7 +21,7 @@ func main() {
        log.Println("Starting pipeline application...")

        http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
-               fmt.Fprintf(w, "Hello world!\n")
+               fmt.Fprintf(w, "Hello hightowerlabs!\n")
        })

        http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
```

Commit the changes and push the `new-message` branch to the `pipeline-application` GitHub repository:

```
git add main.go && git commit -m "change message" && git push origin new-message
```

Pushing a new branch to the `pipeline-application` GitHub repository will trigger the `pipeline-staging-build` build trigger, which will in turn trigger the `pipeline-infrastructure-staging` build trigger:

Review the current builds:

```
gcloud container builds list
```
```
ID                                    CREATE_TIME                DURATION  SOURCE                                  IMAGES                                                                      STATUS
f2f05c4f-e49b-4f7a-bbfb-336dc65ff059  2017-11-17T05:41:09+00:00  12S       pipeline-infrastructure-staging@master  -                                                                           SUCCESS
76db0956-0514-42f5-babb-aad21fdb5689  2017-11-17T05:37:34+00:00  1M2S      pipeline-application@new-message        gcr.io/pipeline-tutorial/pipeline:6e2865a45c29974b0b9099fa824fc00d0128de18  SUCCESS
```

List the container images created by the `pipeline-staging-build` build trigger:

```
gcloud container images list-tags gcr.io/${PROJECT_ID}/pipeline
```
```
DIGEST        TAGS                                      TIMESTAMP
07086bf1e94d  6e2865a45c29974b0b9099fa824fc00d0128de18  2017-11-16T21:37:59
```

List the pods created by the `pipeline-infrastructure-staging` build trigger:

```
kubectl get pods \
  --context gke_${PROJECT_ID}_${COMPUTE_ZONE}_staging
```
```
NAME                        READY     STATUS    RESTARTS   AGE
pipeline-2401200729-tg2hf   1/1       Running   0          3s
```

#### Tag the pipeline-application Repo

In this section you will merge the `new-message` and `master` branches, then create a new tag on the `pipeline-application` GitHub repository, which will trigger a new pipeline container image to be built and deployed to the QA Kubernetes cluster.

Checkout the master branch:

```
git checkout master
```

Merge the `new-message` and `master` branches and push the changes to the `pipeline-applicaiton` GitHub repository:

```
git merge new-message && git push origin master
```

Create a new `1.0.0` tag and push it to the `pipeline-applicaiton` GitHub repository:

```
git tag 1.0.0 && git push origin --tags
```

Pushing a new tag to the `pipeline-applicaiton` GitHub repository will trigger the `pipeline-qa-build` build trigger, which will in turn trigger the `pipeline-infrastructure-qa` build trigger.

Review the current builds:

```
gcloud container builds list
```
```
ID                                    CREATE_TIME                DURATION  SOURCE                                  IMAGES                                                                      STATUS
231edd08-805e-41f7-a2a6-1cd2d25d839b  2017-11-17T06:21:37+00:00  1M1S      pipeline-application@1.0.0              gcr.io/pipeline-tutorial/pipeline:1.0.0                                     SUCCESS
f2f05c4f-e49b-4f7a-bbfb-336dc65ff059  2017-11-17T05:41:09+00:00  12S       pipeline-infrastructure-staging@master  -                                                                           SUCCESS
76db0956-0514-42f5-babb-aad21fdb5689  2017-11-17T05:37:34+00:00  1M2S      pipeline-application@new-message        gcr.io/pipeline-tutorial/pipeline:6e2865a45c29974b0b9099fa824fc00d0128de18  SUCCESS
```

List the container images created by the `pipeline-qa-build` build trigger:

```
gcloud container images list-tags gcr.io/${PROJECT_ID}/pipeline
```
```
DIGEST        TAGS                                      TIMESTAMP
bd6ce000b8ac  1.0.0                                     2017-11-16T22:22:06
07086bf1e94d  6e2865a45c29974b0b9099fa824fc00d0128de18  2017-11-16T21:37:59
```

> Notice a new image was created based on the `1.0.0` tag pushed to the `pipeline-application` GitHub repository.

List the pods created by the `pipeline-qa-deployment` build trigger:

```
kubectl get pods \
  --context gke_${PROJECT_ID}_${COMPUTE_ZONE}_qa
```

```
NAME                       READY     STATUS    RESTARTS   AGE
pipeline-685432654-vsz7h   1/1       Running   0          1m
```

Hit the pipeline application in the QA cluster:

```
PIPELINE_IP_ADDRESS=$(kubectl get svc pipeline \
  --context gke_${PROJECT_ID}_${COMPUTE_ZONE}_qa \
  -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
```

```
curl http://${PIPELINE_IP_ADDRESS}
```

Once the pipeline application is deployed to the QA cluster a pull request is send to the `pipeline-infrastructure-production` GitHub repository. Review and merge the PR on GitHub:

![Image of GitHub UI](images/review-production-pull-request.png)

Merging the pull-request on the `pipeline-infrastructure-production` GitHub repository will trigger the `pipeline-production-deployment` build trigger.

```
gcloud container builds list
```
```
ID                                    CREATE_TIME                DURATION  SOURCE                                     IMAGES                                                                      STATUS
3390166b-0b9e-42e2-ae9a-31a1da90ec3a  2017-11-17T06:35:09+00:00  11S       pipeline-infrastructure-production@master  -                                                                           SUCCESS
1d04e7b4-5252-407d-a1b5-72d3e8ee7512  2017-11-17T06:22:44+00:00  44S       pipeline-infrastructure-qa@master          -                                                                           SUCCESS
231edd08-805e-41f7-a2a6-1cd2d25d839b  2017-11-17T06:21:37+00:00  1M1S      pipeline-application@1.0.0                 gcr.io/pipeline-tutorial/pipeline:1.0.0                                     SUCCESS
f2f05c4f-e49b-4f7a-bbfb-336dc65ff059  2017-11-17T05:41:09+00:00  12S       pipeline-infrastructure-staging@master     -                                                                           SUCCESS
76db0956-0514-42f5-babb-aad21fdb5689  2017-11-17T05:37:34+00:00  1M2S      pipeline-application@new-message           gcr.io/pipeline-tutorial/pipeline:6e2865a45c29974b0b9099fa824fc00d0128de18  SUCCESS
```

List the pods created by the `pipeline-production-deployment` build trigger:

```
kubectl get pods \
  --context gke_${PROJECT_ID}_${COMPUTE_ZONE}_production
```

```
NAME                       READY     STATUS    RESTARTS   AGE
pipeline-685432654-708z9   1/1       Running   0          4s
```

Hit the pipeline application in the production cluster:

```
PIPELINE_IP_ADDRESS=$(kubectl get svc pipeline \
  --context gke_${PROJECT_ID}_${COMPUTE_ZONE}_production \
  -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
```

```
curl http://${PIPELINE_IP_ADDRESS}
```

At this point the `pipeline:1.0.0` container image has been propagated across each environment and is now running in the production Kubernetes cluster.

## Cleanup
