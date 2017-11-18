# Provision Kubernetes Clusters

In this section you will create three Kubernetes clusters using [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine). Each cluster will represent one of the following environments: staging, qa, production.

## Create three Kubernetes Clusters

Create the `staging`, `qa`, and `production` Kubernetes clusters:

```
for e in staging qa production; do
  gcloud container clusters create ${e} --async --num-nodes 1
done
```

It may take up to five minutes to provision the three Kubernetes clusters. Use the `gcloud` command to check the status of each cluster:

```
gcloud container clusters list
```
```
NAME        ZONE        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
production  us-west1-c  1.7.8-gke.0     XX.XXX.XXX.XX  n1-standard-1  1.7.8-gke.0   1          RUNNING
qa          us-west1-c  1.7.8-gke.0     XX.XXX.XX.XX   n1-standard-1  1.7.8-gke.0   1          RUNNING
staging     us-west1-c  1.7.8-gke.0     XX.XXX.XXX.XX  n1-standard-1  1.7.8-gke.0   1          RUNNING
```
