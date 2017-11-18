# pipeline

The pipeline tutorial walks you through creating an end to end deployment pipeline for Kubernetes using [Cloud Container Builder](https://cloud.google.com/container-builder/).

## The Application

This tutorial will set up a pipeline to deploy the [pipeline application](https://github.com/kelseyhightower/pipeline-application), a simple Go application with the following HTTP endpoints:

 * `/` - responds with "Hello world!"
 * `/health` - responds with HTTP status code 200
 * `/version` - responds with the application version (v2.0.0)

## Prerequisites

* [Review the Deployment Pipeline](labs/the-deployment-pipeline.md)
* [Prerequisites](labs/prerequisites.md)

## Tutorial

* [Provision Kubernetes Clusters](labs/kubernetes-clusters.md)
* [Create a Hub Configuration File](labs/hub-configuration-file.md)
* [Setup the Git Repositories](labs/setup-git-repositories.md)
* [Cloud Container Builder Build Triggers](labs/build-triggers.md)
* [Test the Pipeline](labs/test-the-pipeline.md)

## Cleanup
