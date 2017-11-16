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

## Enable Google Cloud Platform APIs

In this section you will enable the GCP APIs required to complete this tutorial.

```
gcloud services enable \
  container.googleapis.com \
  cloudapis.googleapis.com \
  cloudbuild.googleapis.com \
  sourcerepo.googleapis.com \
  compute.googleapis.com \
  storage-component.googleapis.com \
  containerregistry.googleapis.com \
  cloudkms.googleapis.com \
  logging.googleapis.com \
  --async
```

```
gcloud services list --enabled
```
```
NAME                               TITLE
bigquery-json.googleapis.com       BigQuery API
clouddebugger.googleapis.com       Stackdriver Debugger API
datastore.googleapis.com           Google Cloud Datastore API
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
```

### Client Tools

The following client tools are required to complete this tutorial:

 * [hub](https://github.com/github/hub) 2.3.0+
 * [git](https://git-scm.com/downloads) 2.14.0+
 * [gcloud](https://cloud.google.com/sdk) 179.0.0+
 * [kubectl](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.8.md#downloads-for-v183) 1.8.0+

### Create three Kubernetes Clusters

In this section you will create three Kubernetes clusters using Google Container Engine. Each cluster will represent one of the following environments:

 * staging
 * qa
 * production

Create three Kubernetes clusters:

```
gcloud container clusters create staging
```

```
gcloud container clusters create qa
```

```
gcloud container clusters create production
```

### Fork the pipeline repositories

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

#### Fork the pipeline repos

In this section you will fork the following GitHub repos:

* [kelseyhightower/pipeline-application](https://github.com/kelseyhightower/pipeline-application)
* [kelseyhightower/pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging)
* [kelseyhightower/pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa)
* [kelseyhightower/pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production)

Clone and fork the `kelseyhightower/pipeline-application` repo:

```
hub clone https://github.com/kelseyhightower/pipeline-application.git
cd pipeline-application/
hub fork
cd -
```

Clone and fork the `kelseyhightower/pipeline-infrastructure-staging` repo:

```
hub clone https://github.com/kelseyhightower/pipeline-infrastructure-staging.git
cd pipeline-infrastructure-staging/
hub fork
cd -
```

Clone and fork the `kelseyhightower/pipeline-infrastructure-qa` repo:

```
hub clone https://github.com/kelseyhightower/pipeline-infrastructure-qa.git
cd pipeline-infrastructure-qa/
hub fork
cd -
```

Clone and fork the `kelseyhightower/pipeline-infrastructure-production` repo:

```
hub clone https://github.com/kelseyhightower/pipeline-infrastructure-production.git
cd pipeline-infrastructure-production/
hub fork
cd -
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

```
gsutil mb gs://${PROJECT_ID}-pipeline-functions
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

### Create the Cloud Container Builder Build Triggers

First we need to sync the GitHub repos with Google Source Repositories so we can create build triggers.

```
REPOS=(
  pipeline-application
  pipeline-infrastructure-staging
  pipeline-infrastructure-qa
  pipeline-infrastructure-production
)
```

```
for r in ${REPOS[@]}; do
  gcloud source repos create ${r}
  git clone --mirror https://github.com/${GITHUB_USERNAME}/${r}
  git --git-dir ${r}.git push --mirror \
    --repo "https://source.developers.google.com/p/${PROJECT_ID}/r/${r}"
done
```

#### Create the Build Triggers

```
export COMPUTE_ZONE=$(gcloud config get-value compute/zone)
```

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
```

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

Next create the reposync webhook cloud function:

```
gcloud beta functions deploy reposync \
  --entry-point F \
  --stage-bucket ${PROJECT_ID}-pipeline-functions \
  --trigger-http
```
