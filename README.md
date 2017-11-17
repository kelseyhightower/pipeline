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

In this section you will enable the GCP APIs required to complete this tutorial.

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

```
gcloud services list --enabled
```
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
 * [git](https://git-scm.com/downloads) 2.14.0+
 * [gcloud](https://cloud.google.com/sdk) 179.0.0+
 * [kubectl](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.8.md#downloads-for-v183) 1.8.0+

### Create Kubernetes Clusters

In this section you will create three Kubernetes clusters using Google Container Engine. Each cluster will represent one of the following environments:

 * staging
 * qa
 * production

Create three Kubernetes clusters:

```
gcloud container clusters create staging --num-nodes 1
```

```
gcloud container clusters create qa --num-nodes 1
```

```
gcloud container clusters create production --num-nodes 1
```

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

### Create a GitHub API Token

In this section you will fork the pipeline repositories using your GitHub account.

Create a GitHub token using the official [guide](https://github.com/blog/1509-personal-api-tokens)

![Image of GitHub UI](images/create-github-token.png)

> Set the token description to "pipeline" and check the repo scope  

Save the token in the `GITHUB_TOKEN` env var:

```
export GITHUB_TOKEN="<token>"
```

Save your GitHub username in the `GITHUB_USERNAME` env var:

```
export GITHUB_USERNAME="<github-username>"
```

Create a hub configuration file:

```
cat <<EOF > hub-pipeline
github.com:
  - protocol: https
    user: ${GITHUB_USERNAME}
    oauth_token: ${GITHUB_TOKEN}
EOF
```

Set the `HUB_CONFIG` env var to point to the pipeline hub configuration file:

```
HUB_CONFIG="hub-pipeline"
```

### Setup the pipeline repositories

In this section you will fork the following GitHub repos:

* [kelseyhightower/pipeline-application](https://github.com/kelseyhightower/pipeline-application)
* [kelseyhightower/pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging)
* [kelseyhightower/pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa)
* [kelseyhightower/pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production)

```
REPOS=(
  pipeline-application
  pipeline-infrastructure-staging
  pipeline-infrastructure-qa
  pipeline-infrastructure-production
)
```

Clone and fork the pipeline repos:

```
for repo in ${REPOS[@]}; do
  hub clone "https://github.com/kelseyhightower/${repo}.git"
  cd ${repo}/
  hub fork
  cd -
done
```

Clean up the repos:

```
rm -rf \
  pipeline-application \
  pipeline-infrastructure-production \
  pipeline-infrastructure-qa \
  pipeline-infrastructure-staging
```

At this point the pipeline repos have been forked to your GitHub account and can used as part of your own deployment pipeline.

### Save the Hub Credentials to Google Cloud Storage

In this section you will encrypt the hub credentials using the [Google Key Management Service](https://cloud.google.com/kms) (KMS) and upload the encrypted file to a [Google Cloud Storage](https://cloud.google.com/storage) (GCS) bucket.

#### Create a KMS Keyring

Before you can encrypt the hub credentials you need to create a KMS keyring and encryption key.

Create the `pipeline` KMS keyring:

```
gcloud kms keyrings create pipeline --location=global
```

Create the `github` KMS key that will be used to encrypt the hub credentials file:

```
gcloud kms keys create github \
  --location=global \
  --keyring=pipeline \
  --purpose=encryption
```

Encrypt the hub credentials file using the `github` KMS key:

```
gcloud kms encrypt \
  --plaintext-file "${HUB_CONFIG}" \
  --ciphertext-file hub.enc \
  --location=global \
  --keyring=pipeline \
  --key=github
```

#### Copy the encrypted hub credentials file to GCS

In this section you will create a GCS bucket and copy the encrypted hub credentials file to it.

Store the GCP project ID in the `PROJECT_ID` env var:

```
PROJECT_ID=$(gcloud config get-value core/project)
```

Create the pipeline configs GCS bucket:

```
gsutil mb gs://${PROJECT_ID}-pipeline-configs
```

Copy the encrypted hub credentials file to the pipeline configs bucket:

```
gsutil cp hub.enc gs://${PROJECT_ID}-pipeline-configs/
```

#### Grant the Container Builder service account access to the github encryption key

Store the GCP project number in the `PROJECT_NUMBER` env var:

```
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')
```

```
gcloud kms keys add-iam-policy-binding github \
  --location=global \
  --keyring=pipeline \
  --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role=roles/cloudkms.cryptoKeyEncrypterDecrypter
```

Grant the Container Builder service account access to Google Container Engine:

```
gcloud projects add-iam-policy-binding ${PROJECT_NUMBER} \
  --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role=roles/container.developer
```

### Mirror GitHub Repositories to Cloud Source Repositories

Currently Container Builder only supports build triggers on Cloud Source Repositories. In this section you will create Cloud Source Repositories that will mirror each of the GitHub pipeline repositories.

```
REPOS=(
  pipeline-application
  pipeline-infrastructure-staging
  pipeline-infrastructure-qa
  pipeline-infrastructure-production
)
```

For each GitHub repositories create a Cloud Source Repository to mirror it:

```
for r in ${REPOS[@]}; do
  gcloud source repos create ${r}
  git clone --mirror https://github.com/${GITHUB_USERNAME}/${r}
  git --git-dir ${r}.git push --mirror \
    --repo "https://source.developers.google.com/p/${PROJECT_ID}/r/${r}"
done
```

At this point the GitHub repositories are mirrored to your Cloud Source Repositories. To keep them in sync deploy the `reposync` Cloud Function to your project:

#### Deploy the reposync Cloud Function

```
wget https://github.com/kelseyhightower/reposync/releases/download/0.0.1/reposync-cloud-function-0.0.1.zip
```

```
unzip reposync-cloud-function-0.0.1.zip
```

```
cd reposync-cloud-function-0.0.1
```

```
gsutil mb gs://${PROJECT_ID}-pipeline-functions
```

```
gcloud beta functions deploy reposync \
  --entry-point F \
  --stage-bucket ${PROJECT_ID}-pipeline-functions \
  --trigger-http
```

```
cd -
```

### Create the GitHub Webhooks

Store the `reposync` webhook URL:

```
WEBHOOK_URL=$(gcloud beta functions describe reposync \
  --format='value(httpsTrigger.url)')
```

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

```
for repo in ${REPOS[@]}; do
  curl -X POST "https://api.github.com/repos/${GITHUB_USERNAME}/${repo}/hooks" \
    -H "Accept: application/vnd.github.v3+json" \
    -u "${GITHUB_USERNAME}:${GITHUB_TOKEN}" \
    --data-binary @github-webhook-config.json
done
```

### Create the Cloud Builder Triggers

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

### Test the Pipeline

Ensure you have the cluster credentials for each Kubernetes cluster:

```
gcloud container clusters get-credentials staging
```

```
gcloud container clusters get-credentials qa
```

```
gcloud container clusters get-credentials production
```

Ensure no pods are currently running in the default namespace in the staging cluster:

```
kubectl get pods \
  --context gke_${PROJECT_ID}_${COMPUTE_ZONE}_staging
```

Ensure no pods are currently running in the default namespace in the qa cluster:

```
kubectl get pods \
  --context gke_${PROJECT_ID}_${COMPUTE_ZONE}_qa
```

Ensure no pods are currently running in the default namespace in the production cluster:

```
kubectl get pods \
  --context gke_${PROJECT_ID}_${COMPUTE_ZONE}_production
```

#### Update the pipeline-application

In this section you will update the pipeline application and push the changes to a new branch on your pipeline-application GitHub repository.

```
git config --global hub.protocol https
```

```
git config --global credential.https://github.com.helper /usr/local/bin/hub-credential-helper
```


Clone the `pipeline-application` GitHub repository:

```
hub clone ${GITHUB_USERNAME}/pipeline-application
```

```
cd pipeline-application
```

```
git checkout -b new-message
```

Change the message:

```
sed "s/world/${GITHUB_USERNAME}/g" main.go > main.go.new
```

```
mv main.go.new main.go
```

Review the changes:

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

Commit the changes to the `pipeline-application` GitHub repository:

```
git add .
```

```
git commit -m "change message"
```

```
git push origin new-message
```

Review the current builds:

```
gcloud container builds list
ID                                    CREATE_TIME                DURATION  SOURCE                                  IMAGES                                                                      STATUS
f2f05c4f-e49b-4f7a-bbfb-336dc65ff059  2017-11-17T05:41:09+00:00  12S       pipeline-infrastructure-staging@master  -                                                                           SUCCESS
76db0956-0514-42f5-babb-aad21fdb5689  2017-11-17T05:37:34+00:00  1M2S      pipeline-application@new-message        gcr.io/pipeline-tutorial/pipeline:6e2865a45c29974b0b9099fa824fc00d0128de18  SUCCESS
```

List the container images:

```
gcloud container images list --repository gcr.io/${PROJECT_ID}
```
```
NAME
gcr.io/${PROJECT_ID}/pipeline
```

List the container image tags:

```
gcloud container images list-tags gcr.io/${PROJECT_ID}/pipeline
```
```
DIGEST        TAGS                                      TIMESTAMP
07086bf1e94d  6e2865a45c29974b0b9099fa824fc00d0128de18  2017-11-16T21:37:59
```

List the pods in the staging cluster:

```
kubectl get pods \
  --context gke_${PROJECT_ID}_${COMPUTE_ZONE}_staging
```
```
NAME                        READY     STATUS    RESTARTS   AGE
pipeline-2401200729-tg2hf   1/1       Running   0          3s
```
