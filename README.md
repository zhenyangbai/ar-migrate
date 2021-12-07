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

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com" \
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

7.Deploy the container image to Cloud Run
```
SERVICE_NAME=ar-migrate
SERVICE_ACCOUNT=
PROJECT_ID=$(gcloud config get-value project)
REGION=asia-southeast1
     
gcloud run deploy package-upload \
--image=${REGION}-docker.pkg.dev/${PROJECT_ID}/${CONTAINER_REPO_NAME}/${CONTAINER_NAME}:v1 \
--no-allow-unauthenticated \
--service-account=${SERVICE_ACCOUNT} \
--memory=512Mi \
--no-use-http2 \
--cpu-throttling \
--execution-environment=gen1 \
--platform=managed \
--region=${REGION} \
--project=${PROJECT_ID}
```

8. Create an Eventarc trigger[https://cloud.google.com/eventarc/docs/run/quickstart-storage#trigger-setup]
```
SERVICE_NAME=ar-migrate
PROJECT_NUMBER=$(gcloud projects list --filter="$(gcloud config get-value project)" --format="value(PROJECT_NUMBER)")
BUCKET_NAME=python-repo-bucket
REGION=asia-southeast1

 gcloud eventarc triggers create storage-events-trigger \
     --destination-run-service=${SERVICE_NAME} \
     --destination-run-region=${REGION} \
     --event-filters="type=google.cloud.storage.object.v1.finalized" \
     --event-filters="bucket=${BUCKET_NAME}" \
     --service-account=PROJECT_NUMBER-compute@developer.gserviceaccount.com
```

## Deploying

## 
