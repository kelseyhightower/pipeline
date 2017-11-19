# Cleanup


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
