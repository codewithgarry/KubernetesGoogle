# Deploy to Kubernetes in Google Cloud: Challenge Lab (GSP318)

![Google Cloud](https://img.shields.io/badge/Google_Cloud-4285F4?logo=googlecloud&logoColor=white)
![Lab](https://img.shields.io/badge/Lab-GSP318-34A853)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?logo=jenkins&logoColor=white)

Reference solution for the Google Cloud Skills Boost challenge lab **Deploy to Kubernetes in Google Cloud** (GSP318).

## 📖 Overview

You take a Go application called **valkyrie-app** through a full container delivery workflow:

1. Write a Dockerfile and build the image.
2. Test the container locally.
3. Push the image to Container Registry.
4. Deploy it to an existing GKE cluster.
5. Roll out a new version.
6. Automate deployments with a Jenkins pipeline.

## 🧰 Services used

Docker · Container Registry · Google Kubernetes Engine · Cloud Source Repositories · Jenkins on GKE

## ✅ Prerequisites

```bash
export PROJECT_ID=$(gcloud config get-value project)
export ZONE="us-east1-d"

# Lab marking scripts
gsutil cat gs://cloud-training/gsp318/marking/setup_marking.sh | bash
gcloud source repos clone valkyrie-app
```

## 🚀 Solution

### Task 1: Create the Dockerfile and build the image

```bash
cd ~/valkyrie-app

cat > Dockerfile <<'EOS'
FROM golang:1.10
WORKDIR /go/src/app
COPY source .
RUN go install -v
ENTRYPOINT ["app","-single=true","-port=8080"]
EOS

docker build -t valkyrie-app:v0.0.1 .
cd ~/marking && ./step1.sh
```

### Task 2: Test the image locally

```bash
docker run -d -p 8080:8080 valkyrie-app:v0.0.1
cd ~/marking && ./step2.sh
```

### Task 3: Push the image to Container Registry

```bash
cd ~/valkyrie-app
docker tag valkyrie-app:v0.0.1 gcr.io/$PROJECT_ID/valkyrie-app:v0.0.1
docker push gcr.io/$PROJECT_ID/valkyrie-app:v0.0.1
```

### Task 4: Deploy to the GKE cluster

```bash
sed -i "s#IMAGE_HERE#gcr.io/$PROJECT_ID/valkyrie-app:v0.0.1#g" k8s/deployment.yaml

gcloud container clusters get-credentials valkyrie-dev --zone $ZONE
kubectl create -f k8s/deployment.yaml
kubectl create -f k8s/service.yaml
```

### Task 5: Roll out a new version

Scale out, merge the feature branch, then build and roll out `v0.0.2`.

```bash
kubectl scale deployment valkyrie-dev --replicas=3

git merge origin/kurt-dev
docker build -t gcr.io/$PROJECT_ID/valkyrie-app:v0.0.2 .
docker push gcr.io/$PROJECT_ID/valkyrie-app:v0.0.2

# Point every container in the deployment at the new tag
kubectl get deployment valkyrie-dev -o yaml \
  | sed 's/valkyrie-app:v0.0.1/valkyrie-app:v0.0.2/g' \
  | kubectl apply -f -

kubectl rollout status deployment valkyrie-dev
```

### Task 6: Create a Jenkins pipeline

Stop the local test container first, because Jenkins needs port 8080.

```bash
docker ps -q --filter ancestor=valkyrie-app:v0.0.1 | xargs -r docker kill

export POD_NAME=$(kubectl get pods -l "app.kubernetes.io/component=jenkins-master" \
  -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &

# Jenkins admin password
kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode; echo
```

In Jenkins (Web Preview on port 8080):

1. Add a **Google Service Account from metadata** credential for the project.
2. Create a **Multibranch Pipeline** job pointing at the `valkyrie-app` Cloud Source Repository.

Then push a change to trigger the pipeline:

```bash
sed -i "s/green/orange/g" source/html.go
sed -i "s/YOUR_PROJECT/$PROJECT_ID/g" Jenkinsfile

git config --global user.email "$(gcloud config get-value account)"
git config --global user.name  "student"
git add .
git commit -m "Build pipeline: switch theme to orange"
git push
```

Finally, select **Build Now** on the job in Jenkins.

---

## ⚠️ Notes

- Resource names, zones, regions and ports are generated per lab session. Always use the values shown in **your** lab instructions.
- These steps were written when the lab was live. Google revises labs over time, so the current version may differ.

## 📌 Disclaimer

This repository is a study reference. Attempt the lab on your own first, then compare approaches. It is not affiliated with or endorsed by Google.

## 👤 Author

**Girish Sharma** · Cloud & DevOps Engineer · [GitHub](https://github.com/codewithgarry) · [LinkedIn](https://www.linkedin.com/in/codewithgarry/)
