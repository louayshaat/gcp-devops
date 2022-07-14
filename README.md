###
Ensure you have a GKE cluster created
###


To use this builder, your builder service account will need IAM permissions sufficient for the operations you want to perform. For typical read-only usage, the "Kubernetes Engine Viewer" role is sufficient. To deploy container images on a GKE cluster, the "Kubernetes Engine Developer" role is sufficient. Check the GKE IAM page for details.

Running the following command will give Cloud Build Service Account container.developer role access to your Container Engine clusters:

### Prep work
```

git clone https://github.com/louayshaat/gcp-devops

cd gcp-devops

docker build -t exapp .

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
```
# Need to deploy pod once
docker tag exapp gcr.io/coredemos/exapp

docker push gcr.io/core-demos/exapp

kubectl apply -f deployment.yaml

gcloud builds submit --config=cloudbuild.yaml
kubectl logs -lrun=exapp -f
```

### Example 2 - Setup CICD - Create a repo, and setup a Cloud Build trigger
```
ssh-keygen -t rsa -b 4096 -C "source repo build louays@google.com" -f ~/.ssh/myrepokey -P ''
cat ~/.ssh/myrepokey.pub
```
### Upload key to
https://source.cloud.google.com/user/ssh_keys

### Setup Cloud Build
 * Create GCP source repo
 * Configure authentication over SSH 
 * push config files+code to the repo
```
gcloud source repos create mycode-repo
gcloud source repos describe mycode-repo
git config user.email XXXX@XXXXX.com
cat > ~/.ssh/config <<EOF
Host source.developers.google.com
    HostName source.developers.google.com
    User XXXXX@XXXXXX.altostrat.com
    IdentityFile ~/.ssh/myrepokey
EOF
git remote add google ssh://XXXXXX@XXXXX.altostrat.com@source.developers.google.com:2022/p/core-demos/r/mycode-repo
git add .
git commit -m "files"
git push --all google
```

### Trigger a manual Cloud Build run
```
# Create the cloud build trigger
gcloud beta builds triggers import --source=trigger.yaml --verbosity debug
gcloud beta builds triggers list
```

### Trigger an automated Cloud Build run
 * Edit some code
 * Commit those changes back to the repo
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
