# The Deployment Pipeline

This tutorial will leverage [Cloud Container Builder](https://cloud.google.com/container-builder/) to set up the following pipeline:

 * changes pushed to any branch, except the master branch, on the [pipeline application](https://github.com/kelseyhightower/pipeline-application) repo will trigger a container build and a git commit to the [pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging) repo, which updates the pipeline deployment Kubernetes configuration file with the new container image.
 * changes pushed to the [pipeline-infrastructure-staging](https://github.com/kelseyhightower/pipeline-infrastructure-staging) repo will trigger a rolling deployment to a Kubernetes staging cluster.
 * tags on the [pipeline application](https://github.com/kelseyhightower/pipeline-application) repo will trigger a container build based on the tag name and a git commit to the [pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa) repo, which updates the pipeline deployment Kubernetes configuration file with the new container image.
 * changes pushed to the [pipeline-infrastructure-qa](https://github.com/kelseyhightower/pipeline-infrastructure-qa) repo will trigger a rolling deployment to a Kubernetes QA cluster.
 * successful deployments to QA will trigger a pull request on the [pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production) repo.
 * changes pushed to the master branch on the [pipeline-infrastructure-production](https://github.com/kelseyhightower/pipeline-infrastructure-production) repo will trigger a rolling deployment to a Kubernetes production cluster.

 > The rolling update to the production cluster is gated by a pull request to the pipeline-infrastructure-production repo.
