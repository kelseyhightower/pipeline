# Cleanup

## GCP Resources

Delete the GCP project created in the [prerequisites lab](prerequisites.md#create-a-new-project-and-enable-google-cloud-platform-apis). This will shutdown the compute resources create during this tutorial and schedule the project for deletion.

## Delete GitHub Repositories

In this section you will cleanup the GitHub repositories created during this tutorial.

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
  curl -X DELETE "https://api.github.com/repos/${GITHUB_USERNAME}/${repo}"
done
```
