# my-app-repo
Cloud Build CI/CD Pipeline for Deploying a Container Image to GKE
setting up a CI/CD pipeline using Cloud Build to build a container image from the source code, push it to Google Artifact Registry, and deploy it to a Google Kubernetes Engine (GKE) cluster.

	Create Google Cloud Project 
	Cloud Build API enabled.
	Artifact Registry API enabled.
	Kubernetes Cluster (GKE) set up.
2. Create a GKE Cluster (If Not Already Available)
Using Console:
1.	Navigate to Google Kubernetes Engine in the GCP Console.
2.	Click Create Cluster → Select Standard or Autopilot mode.
3.	Configure: 
o	Region/Zone: Select a location.
o	Nodes: Choose machine type (e.g., e2-medium) and number of nodes.
o	Enable Workload Identity for better authentication.
4.	Click Create and wait for the cluster to be ready.
Using gcloud Command:
gcloud container clusters create my-gke-cluster \
  --zone us-central1-a --num-nodes 3 --enable-ip-alias

3. Create Artifact Registry for Storing Container Images
Using Console:
1.	Navigate to Artifact Registry → Click Create Repository.
2.	Choose Docker as the format.
3.	Set a Repository ID (e.g., my-docker-repo).
4.	Choose Region and click Create.
Using gcloud Command:
gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker --location=us-central1

4. Prepare Source Code and Dockerfile
1. Create a Sample Application
Create a directory and add the source code.
mkdir my-app && cd my-app
Create an app.py (Python Flask example) file:
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello, GKE with Cloud Build!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
2. Create a Dockerfile
Inside my-app/, create a Dockerfile:

# Set the working directory
WORKDIR /app

# Copy application files
COPY app.py /app/

# Install dependencies
RUN pip install flask

# Define the command to run the app
CMD ["python", "app.py"]
________________________________________
5. Push Source Code to Cloud Source Repositories (CSR)
Using Console:
1.	Navigate to Cloud Source Repositories in GCP.
2.	Click Create Repository, name it my-app-repo, and click Create.
3.	Connect your local repository: 
	gcloud source repos clone my-app-repo
	cd my-app-repo
	cp -r ../my-app/* .
	git add .
	git commit -m "Initial commit"
	git push origin main

6. Configure Cloud Build Pipeline
1. Create a cloudbuild.yaml File
Create a cloudbuild.yaml file inside your source repository:
substitutions:
  _TAG: "${SHORT_SHA}"

steps:
  # Step 1: Build Docker Image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:${_TAG}'
      - '.'

  # Step 2: Push Image to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:${_TAG}'

  # Step 3: Deploy to GKE
  - name: 'gcr.io/cloud-builders/kubectl'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        gcloud container clusters get-credentials my-gke-cluster --zone=us-central1-a --project=$PROJECT_ID &&
        kubectl set image deployment/my-app-deployment my-app=us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:${_TAG}
    env:
      - 'CLOUDSDK_COMPUTE_ZONE=us-central1-a'
      - 'CLOUDSDK_CONTAINER_CLUSTER=my-gke-cluster'

images:
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:${_TAG}' 
7. Create a Kubernetes Deployment for GKE
1. Create a deployment.yaml File
Inside your repository, create deployment.yaml:
yaml
CopyEdit
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
2. Deploy to GKE
	kubectl apply -f deployment.yaml
8. Configure Cloud Build Triggers
Using Console:
a)	Go to Cloud Build → Triggers.
b)	Click Create Trigger.
c)	Select Cloud Source Repositories and choose my-app-repo.
d)	Set Trigger Type to Branch Push (main).
e)	Under Build Configuration, select cloudbuild.yaml.
f)	Click Create.

9. Validate Deployment
Once the pipeline runs successfully:
1.	Check Cloud Build Logs to ensure no errors.
2.	Verify Pods in GKE: 
	kubectl get pods
3.	Get the Load Balancer IP: 
	kubectl get services my-app-service
4.	Open the external IP in a browser, and you should see: 
Hello, GKE with Cloud Build!


implemented a CI/CD pipeline using Cloud Build to deploy a containerized application to GKE. 
