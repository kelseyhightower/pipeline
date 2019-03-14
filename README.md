# pipeline

The pipeline tutorial walks you through creating an end-to-end deployment pipeline using [Cloud Build](https://cloud.google.com/cloud-build), [GitHub](https://github.com), and multiple [Kubernetes](https://cloud.google.com/kubernetes-engine) clusters.

This tutorial will demonstrate how to propagate a Kubernetes deployment through multiple environments, each backed by a dedicated Kubernetes cluster, using a collection of Kubernetes manifest files across a set of GitHub repositories representing each environment.

> The use of multiple Kubernetes clusters and GitHub repositories enables fine grained access control for each environment and streamlines automated build steps targeting those environments.

## The Application

This tutorial will set up a pipeline to deploy the [pipeline application](https://github.com/kelseyhightower/pipeline-application), a simple Go application with the following HTTP endpoints:

 * `/` - responds with "Hello world!"
 * `/health` - responds with HTTP status code 200
 * `/version` - responds with the application version (v2.0.0)

## Prerequisites

* [Review the Deployment Pipeline](labs/deployment-pipeline.md)
* [Prerequisites](labs/prerequisites.md)

## Tutorial

* [Provision the Kubernetes Clusters](labs/kubernetes-clusters.md)
* [Create a Hub Configuration File](labs/hub-configuration-file.md)
* [Setup the GitHub Repositories](labs/github-repositories.md)
* [Create the Cloud Build Build Triggers](labs/build-triggers.md)
* [Test the Build Pipeline](labs/test-the-pipeline.md)

## Cleanup

* [Cleaning Up](labs/cleanup.md)
