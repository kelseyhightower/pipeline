# pipeline

Create three GKE clusters:

```
gcloud container clusters create staging
```

```
gcloud container clusters create qa
```

```
gcloud container clusters create production
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

Create a GitHub token: https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line

```
GITHUB_TOKEN="<token>"
```

```
cat <<EOF > hub
github.com:
  - protocol: https
    user: ${GITHUB_USERNAME}
    oauth_token: ${GITHUB_TOKEN}
EOF
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
