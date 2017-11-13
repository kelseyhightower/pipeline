# pipeline

The pipeline tutorial walks you through creating an end to end deployment pipeline for Kubernetes using [Cloud Container Builder](https://cloud.google.com/container-builder/).

## The Application

This tutorial will set up a pipeline to deploy the [pipeline application](https://github.com/kelseyhightower/pipeline-application), a simple Go application with the following HTTP endpoints:

 * `/` - responds with "Hello world!"
 * `/health` - responds with HTTP status code 200
 * `/version` - responds with the application version (v2.0.0)

## The Deployment Pipeline

This tutorial will leverage [Cloud Container Builder](https://cloud.google.com/container-builder/) to set up the following pipeline:

 * changes pushed to any branch except master on the [pipeline application](https://github.com/kelseyhightower/pipeline-application) repo will trigger a container build and a rolling deployment to a Kubernetes staging cluster.
 * tags on the [pipeline application](https://github.com/kelseyhightower/pipeline-application) repo will trigger a container build based on the tag name and a rolling deployment to a Kubernetes QA cluster.
 * successful deployments to QA will trigger a pull request the [pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production) repo.
 * changed pushed to the master branch on the `pipeline-infrastructure-production` repo will trigger a rolling deployment to a Kubernetes production cluster

## Tutorial

### Services

The following services are required to complete this tutorial:

* [GitHub](https://github.com)
* [Google Cloud Platform](https://console.cloud.google.com/freetrial)

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
cat <<EOF > ~/.config/hub-pipeline
github.com:
  - protocol: https
    user: ${GITHUB_USERNAME}
    oauth_token: ${GITHUB_TOKEN}
EOF
```

Set the `HUB_CONFIG` env var to point to the pipeline hub configuration file:

```
HUB_CONFIG="~/.config/hub-pipeline"
```

Fork the pipeline repos:

```
hub clone https://github.com/kelseyhightower/pipeline-application.git
cd pipeline-application/
hub fork
cd -
```

```
hub clone https://github.com/kelseyhightower/pipeline-infrastructure-staging.git
cd pipeline-infrastructure-staging/
hub fork
cd -
```

```
hub clone https://github.com/kelseyhightower/pipeline-infrastructure-qa.git
cd pipeline-infrastructure-qa/
hub fork
cd -
```

```
hub clone https://github.com/kelseyhightower/pipeline-infrastructure-production.git
cd pipeline-infrastructure-production/
hub fork
cd -
```

Create the pipeline keyring:

```
gcloud kms keyrings create pipeline --location=global
```

```
gcloud kms keys create github \
  --location=global \
  --keyring=pipeline \
  --purpose=encryption
```

```
gcloud kms encrypt \
  --plaintext-file hub \
  --ciphertext-file hub.enc \
  --location=global \
  --keyring=pipeline \
  --key=github
```

Make a GCS bucket

```
PROJECT_ID=$(gcloud config get-value core/project)
```

```
gsutil mb gs://${PROJECT_ID}-pipeline-configs
```

```
gsutil cp hub.enc gs://${PROJECT_ID}-pipeline-configs/
```


```
[^(?!.*master)].*
```
