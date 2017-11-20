# The Deployment Pipeline

The heavy lifting of the deployment pipeline is done by [Cloud Container Builder](https://cloud.google.com/container-builder/) and [Kubernetes](https://cloud.google.com/kubernetes-engine).

Take a moment to study the Container Builder build requests files (cloudbuild.yaml) across the various GitHub repositories to get a sense of how the pipeline fits together:

* [pipeline-application](https://github.com/kelseyhightower/pipeline-application)
* [pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging)
* [pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa)
* [pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production)

## High Level Diagram

Coming soon.

## High Level Text Description

 * changes pushed to any branch, except the master branch, on the [pipeline application](https://github.com/kelseyhightower/pipeline-application) repo will trigger a container build and a git commit to the [pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging) repo, which updates the pipeline deployment Kubernetes configuration file with the new container image.
 * changes pushed to the [pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging) repo will trigger a rolling deployment to a Kubernetes staging cluster.
 * tags on the [pipeline application](https://github.com/kelseyhightower/pipeline-application) repo will trigger a container build based on the tag name and a git commit to the [pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa) repo, which updates the pipeline deployment Kubernetes configuration file with the new container image.
 * changes pushed to the [pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa) repo will trigger a rolling deployment to a Kubernetes QA cluster.
 * successful deployments to QA will trigger a pull request on the [pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production) repo.
 * changes pushed to the master branch on the [pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production) repo will trigger a rolling deployment to a Kubernetes production cluster.

 > The rolling update to the production cluster is gated by a pull request to the pipeline-infrastructure-production repo.
