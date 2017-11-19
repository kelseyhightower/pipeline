# Provision Kubernetes Clusters

In this section you will create a Kubernetes cluster using [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) for each of the following environments:

* staging
* qa
* production

Using dedicated Kubernetes clusters provides better isolation between environments which can result in simpler cluster configurations.

## Create the Kubernetes Clusters

Set the list of environments:

```
ENVIRONMENTS=(
  staging
  qa
  production
)
```

Create a Kubernetes clusters for each environment:

```
for e in ${ENVIRONMENTS[@]}; do
  gcloud container clusters create ${e} --async --num-nodes 1
done
```

It can take up to five minutes to provision the Kubernetes clusters. Use the `gcloud` command to check the status of each cluster:

```
gcloud container clusters list
```
```
NAME        ZONE        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
production  us-west1-c  1.7.8-gke.0     XX.XXX.XXX.XX  n1-standard-1  1.7.8-gke.0   1          RUNNING
qa          us-west1-c  1.7.8-gke.0     XX.XXX.XX.XX   n1-standard-1  1.7.8-gke.0   1          RUNNING
staging     us-west1-c  1.7.8-gke.0     XX.XXX.XXX.XX  n1-standard-1  1.7.8-gke.0   1          RUNNING
```

## Grant the Container Builder Service Account access to the Google Kubernetes Engine API

The Container Builder service needs the ability to fetch GKE credentials and apply changes to each Kubernetes cluster. Grant the Container Builder service account developer access to the GKE API.

```
gcloud projects add-iam-policy-binding ${PROJECT_NUMBER} \
  --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role=roles/container.developer
```

Next: [Create a Hub Configuration File](hub-configuration-file.md)
