# ar-migrate
Cloud Run Job to Sync Python Packages from GCS Bucket to Artifact Registry using Eventarc

## Architecture 
![alt text](https://raw.githubusercontent.com/zhenyangbai/ar-migrate/main/blob/Artifact%20Registry.png)

## Setup

1. [Create a Python Repository](https://cloud.google.com/artifact-registry/docs/python/quickstart#create)

```
PYTHON_REPO_NAME=python-repo
REGION=asia-southeast1

gcloud artifacts repositories create ${PYTHON_REPO_NAME}
    --repository-format=python \
    --location=${REGION} \
    --description="Python package repository"
```

2. [Create a Docker Repository](https://cloud.google.com/artifact-registry/docs/docker/quickstart#create)

```
CONTAINER_REPO_NAME=docker-registry
REGION=asia-southeast1

gcloud artifacts repositories create ${CONTAINER_REPO_NAME}
    --repository-format=python \
    --location=${REGION} \
    --description="Docker repository"
```

3. [Create a Storage Bucket](https://cloud.google.com/eventarc/docs/run/quickstart-storage#create-bucket)
```
BUCKET_NAME=python-repo-bucket
REGION=asia-southeast1

gsutil mb -l ${REGION} gs://${BUCKET_NAME}/
```

4. Grant the pubsub.publisher role to the Cloud Storage service account
```
PROJECT_ID=$(gcloud config get-value project)
SERVICE_ACCOUNT="$(gsutil kms serviceaccount -p ${PROJECT_ID})"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:${SERVICE_ACCOUNT}" \
    --role='roles/pubsub.publisher'
```

5. If you enabled the Pub/Sub service account on or before April 8, 2021, grant the iam.serviceAccountTokenCreator role to the Pub/Sub service account
```
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects list --filter="$(gcloud config get-value project)" --format="value(PROJECT_NUMBER)")

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com" \
    --role='roles/iam.serviceAccountTokenCreator'
```

6. Build the container and upload it to Cloud Build
```
REGION=asia-southeast1
PROJECT_ID=$(gcloud config get-value project)
CONTAINER_REPO_NAME=docker-registry
CONTAINER_NAME=ar-migrate

git clone https://github.com/zhenyangbai/ar-migrate.git
gcloud builds submit --tag $REGION-docker.pkg.dev/${PROJECT_ID}/${CONTAINER_REPO_NAME}/${CONTAINER_NAME}:v1
```

7. [Create a Cloud Run Service Account for artifact registry](https://cloud.google.com/artifact-registry/docs/access-control#grant-repo)
```
RUN_SERVICE_ACCOUNT=run-artifact
RUN_ROLE_NAME="roles/artifactregistry.writer"
PYTHON_REPO_NAME=python-repo
REGION=asia-southeast1
PROJECT_ID=$(gcloud config get-value project)

gcloud iam service-accounts create ${RUN_SERVICE_ACCOUNT} \
    --description="Cloud Run Service Account for Artifact Registry" \
    --display-name="Cloud Run Service Account for Artifact Registry"

gcloud artifacts repositories add-iam-policy-binding ${PYTHON_REPO_NAME} \
    --location ${REGION} \
    --member="serviceAccount:${RUN_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role=${RUN_ROLE_NAME}
```


8. Deploy the container image to Cloud Run
```
RUN_SERVICE_ACCOUNT=run-artifact
SERVICE_NAME=ar-migrate
PROJECT_ID=$(gcloud config get-value project)
REGION=asia-southeast1
CONTAINER_REPO_NAME=docker-registry
CONTAINER_NAME=ar-migrate

gcloud run deploy ${SERVICE_NAME} \
--image=${REGION}-docker.pkg.dev/${PROJECT_ID}/${CONTAINER_REPO_NAME}/${CONTAINER_NAME}:v1 \
--no-allow-unauthenticated \
--service-account=${RUN_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com \
--memory=512Mi \
--no-use-http2 \
--cpu-throttling \
--execution-environment=gen1 \
--platform=managed \
--region=${REGION} \
--project=${PROJECT_ID}
```

7. [Create a EventArc Service Account for invoking Cloud Run](https://cloud.google.com/run/docs/securing/managing-access)
```
SERVICE_NAME=ar-migrate
TRIGGER_SERVICE_ACCOUNT=trigger-run
TRIGGER_ROLE_NAME="roles/run.invoker"
PROJECT_ID=$(gcloud config get-value project)

gcloud iam service-accounts create ${TRIGGER_SERVICE_ACCOUNT} \
    --description="EventArc Service Account for Triggering Cloud Run" \
    --display-name="EventArc Service Account for Triggering Cloud Run"
    
gcloud run services add-iam-policy-binding ${SERVICE_NAME} \
    --member="serviceAccount:${TRIGGER_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role=${TRIGGER_ROLE_NAME}
```

10. [Create an Eventarc trigger](https://cloud.google.com/eventarc/docs/run/quickstart-storage#trigger-setup)
```
TRIGGER_SERVICE_NAME=ar-migrate
BUCKET_NAME=python-repo-bucket
REGION=asia-southeast1
TRIGGER_SERVICE_ACCOUNT=trigger-run
PROJECT_ID=$(gcloud config get-value project)

gcloud eventarc triggers create storage-events-trigger \
     --destination-run-service=${SERVICE_NAME} \
     --destination-run-region=${REGION} \
     --event-filters="type=google.cloud.storage.object.v1.finalized" \
     --event-filters="bucket=${BUCKET_NAME}" \
     --service-account="${TRIGGER_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com"
```
