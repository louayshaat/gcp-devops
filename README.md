### Pre-requisites
Ensure you have a GKE cluster created
###

### Permissions
To use this builder, your builder service account will need IAM permissions sufficient for the operations you want to perform. For typical read-only usage, the "Kubernetes Engine Viewer" role is sufficient. To deploy container images on a GKE cluster, the "Kubernetes Engine Developer" role is sufficient. Check the GKE IAM page for details.


### Prep work
```
gcloud config set project core-demos

gcloud services enable containerregistry.googleapis.com

gcloud services enable cloudbuild.googleapis.com

gcloud container clusters get-credentials --zone=us-central1-c devops-demo

PROJECT="$(gcloud projects describe \
    $(gcloud config get-value core/project -q) --format='get(projectNumber)')"

gcloud projects add-iam-policy-binding $PROJECT \
    --member=serviceAccount:$PROJECT@cloudbuild.gserviceaccount.com \
    --role=roles/container.developer
```

### Example 1 - Manually submit a Cloud Build job

#### Clone the repo
```
git clone https://github.com/louayshaat/gcp-devops
cd gcp-devops
```

#### Build and push the image
```
docker build -t exapp .

docker tag exapp gcr.io/coredemos/exapp

docker push gcr.io/core-demos/exapp
```
#### Deploy to GKE
```
kubectl apply -f deployment.yaml

gcloud builds submit --config=cloudbuild.yaml
```
#### Watch the logs
```
kubectl logs -lrun=exapp -f
```

### Setup CICD - Create a repo, and setup a Cloud Build trigger

#### Create your ssh keys
```
ssh-keygen -t rsa -b 4096 -C "source repo build louays@google.com" -f ~/.ssh/myrepokey -P ''
cat ~/.ssh/myrepokey.pub
```
### Upload key to GCP

https://source.cloud.google.com/user/ssh_keys

### Setup Cloud Build


#### Create GCP source repo
```
gcloud source repos create mycode-repo
gcloud source repos describe mycode-repo
git config user.email XXXX@XXXXX.com
```
#### Configure authentication over SSH
```
cat > ~/.ssh/config <<EOF
Host source.developers.google.com
    HostName source.developers.google.com
    User XXXXX@XXXXXX.altostrat.com
    IdentityFile ~/.ssh/myrepokey
EOF
```

#### Push config  to the repo
```
git remote add google ssh://XXXXXX@XXXXX.altostrat.com@source.developers.google.com:2022/p/core-demos/r/mycode-repo
git add .
git commit -m "files"
git push --all google
```

#### Trigger a manual Cloud Build run
```
# Create the cloud build trigger
gcloud beta builds triggers import --source=trigger.yaml --verbosity debug
gcloud beta builds triggers list
```

### Trigger an automated Cloud Build run


#### Edit some code & commit it
```
vim main.go
git add .
git commit -m "update 1"
git push --all google
watch -t -n2 kubectl logs -lrun=exapp
kubectl get events --sort-by='{.lastTimestamp}' --namespace=default --watch
```

### Goto the following services in the portal
 * Cloud Build History
 * Container Registry
 * Cloud Source Repositories
