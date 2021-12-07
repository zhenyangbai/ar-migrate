# ar-migrate
Cloud Run Job to Sync Python Packages from GCS Bucket to Artifact Registry using Eventarc

## Architecture 
![alt text](https://raw.githubusercontent.com/zhenyangbai/ar-migrate/main/blob/GCS%20-_%20Artifact%20Registry.png)


## Enabling Google APIs
```
gcloud services enable \
  artifactregistry.googleapis.com \
  storage.googleapis.com \
  pubsub.googleapis.com \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  eventarc.googleapis.com \
  logging.googleapis.com
```

## Setup Up Env Variables
```
export REGION=asia-southeast1
export PYTHON_REPO_NAME=python-repo
export CONTAINER_REPO_NAME=docker-registry
export CONTAINER_NAME=ar-migrate
export BUCKET_NAME=python-repo-bucket
export REGION=asia-southeast1
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects list --filter="$(gcloud config get-value project)" --format="value(PROJECT_NUMBER)")
export STORAGE_SERVICE_ACCOUNT="$(gsutil kms serviceaccount -p ${PROJECT_ID})"
export SERVICE_NAME=ar-migrate
export RUN_SERVICE_ACCOUNT=run-artifact
export RUN_ROLE_NAME="roles/artifactregistry.writer"
export TRIGGER_SERVICE_ACCOUNT=trigger-run
export TRIGGER_ROLE_NAME="roles/run.invoker"
```

## Setup Repositories and Storage Bucket
1. [Create a Python Repository](https://cloud.google.com/artifact-registry/docs/python/quickstart#create)

```
gcloud artifacts repositories create ${PYTHON_REPO_NAME} \
    --repository-format=python \
    --location=${REGION} \
    --description="Python package repository"
```

2. [Create a Docker Repository](https://cloud.google.com/artifact-registry/docs/docker/quickstart#create)

```

gcloud artifacts repositories create ${CONTAINER_REPO_NAME} \
    --repository-format=docker \
    --location=${REGION} \
    --description="Docker repository"
```

3. [Create a Storage Bucket](https://cloud.google.com/eventarc/docs/run/quickstart-storage#create-bucket)
```
gsutil mb -l ${REGION} gs://${BUCKET_NAME}/
```

## Setup Service Accounts and Roles
1. Grant the pubsub.publisher role to the Cloud Storage service account
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:${STORAGE_SERVICE_ACCOUNT}" \
    --role='roles/pubsub.publisher'
```

2. If you enabled the Pub/Sub service account on or before April 8, 2021, grant the iam.serviceAccountTokenCreator role to the Pub/Sub service account
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com" \
    --role='roles/iam.serviceAccountTokenCreator'
```

3. [Create a Cloud Run Service Account for Artifact Registry and Storage Bucket](https://cloud.google.com/artifact-registry/docs/access-control#grant-repo)
```
gcloud iam service-accounts create ${RUN_SERVICE_ACCOUNT} \
    --description="Cloud Run Service Account for Artifact Registry" \
    --display-name="Cloud Run Service Account for Artifact Registry"

gcloud artifacts repositories add-iam-policy-binding ${PYTHON_REPO_NAME} \
    --location ${REGION} \
    --member="serviceAccount:${RUN_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role=${RUN_ROLE_NAME}
    
gsutil iam ch serviceAccount:${RUN_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com:legacyBucketReader gs://${BUCKET_NAME}
gsutil iam ch serviceAccount:${RUN_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com:objectViewer gs://${BUCKET_NAME}
```

4. [Create a EventArc Service Account for invoking Cloud Run](https://cloud.google.com/run/docs/securing/managing-access)
```
gcloud iam service-accounts create ${TRIGGER_SERVICE_ACCOUNT} \
    --description="EventArc Service Account for Triggering Cloud Run" \
    --display-name="EventArc Service Account for Triggering Cloud Run"
```

## Deploying
1. Build the container and upload it to Cloud Build
```
git clone https://github.com/zhenyangbai/ar-migrate.git
cd ar-migrate
gcloud builds submit --tag $REGION-docker.pkg.dev/${PROJECT_ID}/${CONTAINER_REPO_NAME}/${CONTAINER_NAME}:v1
```

2. Deploy the container image to Cloud Run
```
gcloud run deploy ar-migrate \
    --image=${REGION}-docker.pkg.dev/${PROJECT_ID}/${CONTAINER_REPO_NAME}/${CONTAINER_NAME}:v1 \
    --no-allow-unauthenticated \
    --service-account=${RUN_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com \
    --set-env-vars=PROJECT=${PROJECT_ID},BUCKET=${BUCKET_NAME},REGION=${REGION},REGISTRY_NAME=${PYTHON_REPO_NAME} \
    --no-use-http2 \
    --binary-authorization default \
    --platform=managed \
    --region=${REGION} \
    --project=${PROJECT_ID}
```

3. Enable EventArc Service Account to Invoke the specific Cloud Run Service
```
gcloud run services add-iam-policy-binding ${SERVICE_NAME} \
    --member="serviceAccount:${TRIGGER_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role=${TRIGGER_ROLE_NAME}
```

3. [Create an Eventarc trigger](https://cloud.google.com/eventarc/docs/run/quickstart-storage#trigger-setup)
```
gcloud eventarc triggers create storage-events-trigger \
     --location=${REGION} \
     --destination-run-service=${SERVICE_NAME} \
     --destination-run-region=${REGION} \
     --event-filters="type=google.cloud.storage.object.v1.finalized" \
     --event-filters="bucket=${BUCKET_NAME}" \
     --service-account="${TRIGGER_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com"
```
