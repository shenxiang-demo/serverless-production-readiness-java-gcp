# Books GenAI - Cloud Run Service using SpringAI, VertexAI, Gemini

This lab can be executed directly in a Cloud Workstation, Cloudshell or your environment of choice.

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.svg)](https://shell.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https://github.com/GoogleCloudPlatform/serverless-photosharing-workshop.git)

## Setup Java ecosystem
In order to build JIT or Native Java app images, please set up Java and GraalVM and the associated Java 21 distributions.
A simple installer is available from [SDKMan](https://sdkman.io/install)
```shell
sdk install java 21.0.2-graal
```

## Clone the code:
```shell
git clone https://github.com/GoogleCloudPlatform/serverless-production-readiness-java-gcp.git
cd serverless-production-readiness-java-gcp/sessions/next24/books-genai-vertex-springai 
```

## Install the maven wrapper
The Maven Wrapper is an easy way to ensure a user of your Maven build has everything necessary to run your Maven build.

Run the command:
```shell
mvn wrapper:wrapper
```

## Build the service code and publish images to the container registry
Build the JIT app image:
```shell
#wWith tests:
./mvnw package

# without tests:
./mvnw package -DskipTests

Start the app locally requires the ability to reach alloydb from your machine:

export DB_URL=jdbc:postgresql://alloy-ipaddress:port/library
export MY_USER=user
export MY_PASSWORD=pword 

java -jar target/books-genai-1.0.0.jar 
  or
./mvnw spring-boot:run
```

Build the Native Java executable:
```shell
# with tests
./mvnw native:compile -Pnative

# without tests
./mvnw native:compile -Pnative -DskipTests

Test the executable locally:
./target/books-genai
```

### Build a JIT and Native Java container Image
```shell
./mvnw spring-boot:build-image -Dskiptests -Dspring-boot.build-image.imageName=books-genai-jit

./mvnw spring-boot:build-image -Pnative -DskipTests -Dspring-boot.build-image.imageName=books-genai-native
```

Check the Docker image sizes:
```shell
docker images | grep books-genai
books-genai-native                                     latest                aa0f7b406966   44 years ago    372MB
books-genai-jit                                        latest                67b3489cbec8   44 years ago    467MB
```

Start the Docker images locally. The image naming conventions indicate whether the image was built by Maven|Gradle and contains the JIT|NATIVE version
```shell
docker run --rm books-genai-jit
docker run --rm books-genai-native
```

Retrieve the Project ID, as it will be required for the next GCP operations
```shell
export PROJECT_ID=$(gcloud config get-value project)
echo $PROJECT_ID
```

Tag and push the images to GCR or Artifact Registry:
```shell
# Container Registry
docker tag books-genai-jit gcr.io/${PROJECT_ID}/books-genai-jit
docker tag books-genai-native gcr.io/${PROJECT_ID}/books-genai-native

docker push gcr.io/${PROJECT_ID}/books-genai-jit
docker push gcr.io/${PROJECT_ID}/books-genai-native

# Artifact Registry
docker tag books-genai-jit:latest us-docker.pkg.dev/${PROJECT_ID}/books-genai-jit/books-genai:latest
docker tag books-genai-native:latest us-docker.pkg.dev/${PROJECT_ID}/books-genai-native/books-genai:latest

docker push us-docker.pkg.dev/${PROJECT_ID}/books-genai-jit/books-genai:latest 
docker push us-docker.pkg.dev/${PROJECT_ID}/books-genai-native/books-genai:latest 
```

## Deploy and run workshop code

Enable the required APIs, if they are not already enabled:
```shell
gcloud services enable vision.googleapis.com
gcloud services enable cloudfunctions.googleapis.com
gcloud services enable cloudbuild.googleapis.com 
gcloud services enable run.googleapis.com
gcloud services enable logging.googleapis.com 
gcloud services enable storage-component.googleapis.com 
gcloud services enable aiplatform.googleapis.com
```

Retrieve the Project ID and Project Number
```shell
# get the Project_ID
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
  or 
export PROJECT_ID=$(gcloud config get-value project)

# get the Project_Number
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
```

## Create a vpc
Create a vpc called default VPC with subnet in us-central1  [here](https://cloud.google.com/vpc/docs/create-modify-vpc-networks)

## Create the GCS bucket
```shell
export BUCKET_PICTURES=library_next24_images
gsutil mb -l us-central1 gs://${BUCKET_PICTURES}
gsutil uniformbucketlevelaccess set on gs://${BUCKET_PICTURES}
gsutil iam ch allUsers:objectViewer gs://${BUCKET_PICTURES}

export BUCKET_BOOKS_PUBLIC=libarary_next24_public
gsutil mb -l us-central1 gs://${BUCKET_BOOKS_PUBLIC}
gsutil uniformbucketlevelaccess set on gs://${BUCKET_BOOKS_PUBLIC}
gsutil iam ch allUsers:objectViewer gs://${BUCKET_BOOKS_PUBLIC}

export BUCKET_BOOKS_PRIVATE=libarary_next24_private
gsutil mb -l us-central1 gs://${BUCKET_BOOKS_PRIVATE}
gsutil uniformbucketlevelaccess set on gs://${BUCKET_BOOKS_PRIVATE}
gsutil iam ch allUsers:objectViewer gs://${BUCKET_BOOKS_PRIVATE}
```

## Create the database
Instructions for configuring cloud Firestore available [here](https://codelabs.developers.google.com/codelabs/cloud-picadaily-lab1?hl=en&continue=https%3A%2F%2Fcodelabs.developers.google.com%2Fserverless-workshop%2F#8)

## Create the alloydb database
Instructions for configuring cloud AlloyDB available [here](https://codelabs.developers.google.com/codelabs/alloydb-ai-embedding)
Once the AlloyDB instance is up and Vertex AI is enabled. Skip creating quickstart_db and import data.

Create a new database called library instead:
```shell
psql "host=$INSTANCE_IP user=postgres" -c "CREATE DATABASE library"
````
Then run the following DDL statements to create the tables:
[DDL-Library](sql/books-ddl.sql)


Set config variables
```shell
gcloud config set project ${PROJECT_ID}
gcloud config set run/region 
gcloud config set run/platform managed
gcloud config set eventarc/location us-central1
```

Grant `pubsub.publisher` to Cloud Storage service account
Grant permission to generate and send events to configured eventarc riggers and ability to invoke google cloud run services
Grant permission to compute sa used by cloud run to accept events, ability to access the cloud storage, and vertex ai apis
```shell
SERVICE_ACCOUNT="$(gsutil kms serviceaccount -p ${PROJECT_ID})"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:${SERVICE_ACCOUNT}" \
    --role='roles/pubsub.publisher'
    
gcloud projects add-iam-policy-binding next24-genai-app \
    --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-eventarc.iam.gserviceaccount.com" \
    --role="roles/eventarc.serviceAgent"

gcloud projects add-iam-policy-binding next24-genai-app \
    --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-eventarc.iam.gserviceaccount.com" \
    --role="roles/run.invoker"

gcloud projects add-iam-policy-binding next24-genai-app \
    --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
    --role="roles/eventarc.eventReceiver"
gcloud projects add-iam-policy-binding next24-genai-app \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/aiplatform.user"
gcloud projects add-iam-policy-binding next24-genai-app \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/datastore.user" 
```

Create VPC connectors for Cloud Run to connect to alloy
```shell
# deploy JIT image to Cloud Run
gcloud compute networks vpc-access connectors create alloy-connector \
--region us-central1 \
--network default \
--range 10.100.0.0/28
````

Deploy to Cloud Run
```shell
# deploy JIT image to Cloud Run
gcloud run deploy books-genai-jit \
  --set-env-vars='MY_PASSWORD=<password>,MY_USER=<user>,DB_URL=<DB_URL>,VERTEX_AI_GEMINI_PROJECT_ID=${$PROJECT_ID},VERTEX_AI_GEMINI_LOCATION=us-central1' \
  --image us-docker.pkg.dev/${$PROJECT_ID}/books-genai-jit/books-genai:latest  --region us-central1 \
  --memory 4Gi --cpu 4 --cpu-boost --execution-environment=gen2  \
  --set-env-vars=JAVA_TOOL_OPTIONS='-XX:+UseZGC -XX:+ZGenerational -XX:MaxRAMPercentage=75 -XX:ActiveProcessorCount=4 -XX:+TieredCompilation -XX:TieredStopAtLevel=1 -Xss256k' \
  --allow-unauthenticated \
  --vpc-connector alloy-connector

# deploy native Java image to Cloud Run
gcloud run deploy books-genai-native \
  --set-env-vars='MY_PASSWORD=<password>,MY_USER=<user>,DB_URL=<DB_URL>,VERTEX_AI_GEMINI_PROJECT_ID=${$PROJECT_ID},VERTEX_AI_GEMINI_LOCATION=us-central1' \
  --image us-docker.pkg.dev/${$PROJECT_ID}/books-genai-native/books-genai:latest  --region us-central1 \
  --memory 4Gi --cpu 4 --cpu-boost --execution-environment=gen2  \
  --set-env-vars=JAVA_TOOL_OPTIONS='-XX:+UseG1GC -XX:MaxRAMPercentage=75 -XX:ActiveProcessorCount=4 -XX:+TieredCompilation -XX:TieredStopAtLevel=1 -Xss256k' \
  --allow-unauthenticated \
  --vpc-connector alloy-connector
```

Set up Eventarc triggers
```shell
gcloud eventarc triggers list --location=us-central1

# configure triggers for public and private books, images - accessing the Native Java service image
gcloud eventarc triggers create books-genai-jit-trigger-image \
     --destination-run-service=books-genai-jit \
     --destination-run-region=us-central1 \
     --destination-run-path=/images \
     --location=us-central1 \
     --event-filters="type=google.cloud.storage.object.v1.finalized" \
     --event-filters="bucket=vision-${PROJECT_ID}" \
     --service-account=${PROJECT_NUMBER}-compute@developer.gserviceaccount.com

gcloud eventarc triggers create books-genai-native-trigger-image \
     --destination-run-service=books-genai-native \
     --destination-run-region=us-central1 \
     --destination-run-path=/images \
     --location=us-central1 \
     --event-filters="type=google.cloud.storage.object.v1.finalized" \
     --event-filters="bucket=vision-${PROJECT_ID}" \
     --service-account=${PROJECT_NUMBER}-compute@developer.gserviceaccount.com

gcloud eventarc triggers create books-genai-jit-trigger-embeddings \
     --destination-run-service=books-genai-jit \
     --destination-run-region=us-central1 \
     --destination-run-path=/document/embeddings \
     --location=us-central1 \
     --event-filters="type=google.cloud.storage.object.v1.finalized" \
     --event-filters="bucket=books-${PROJECT_ID}" \
     --service-account=48099017975-compute@developer.gserviceaccount.com

# configure triggers for public and private books, images - accessing the Native Java service image
gcloud eventarc triggers create books-genai-native-trigger-public \
     --destination-run-service=books-genai-native \
     --destination-run-region=us-central1 \
     --destination-run-path=/document/embeddings \
     --location=us-central1 \
     --event-filters="type=google.cloud.storage.object.v1.finalized" \
     --event-filters="bucket=library_next24_public" \
     --service-account=48099017975-compute@developer.gserviceaccount.com

gcloud eventarc triggers create books-genai-native-trigger-private \
     --destination-run-service=books-genai-native \
     --destination-run-region=us-central1 \
     --destination-run-path=/document/embeddings \
     --location=us-central1 \
     --event-filters="type=google.cloud.storage.object.v1.finalized" \
     --event-filters="bucket=library_next24_private" \
     --service-account=48099017975-compute@developer.gserviceaccount.com

gcloud eventarc triggers create books-genai-native-trigger-image \
     --destination-run-service=books-genai-native \
     --destination-run-region=us-central1 \
     --destination-run-path=/images \
     --location=us-central1 \
     --event-filters="type=google.cloud.storage.object.v1.finalized" \
     --event-filters="bucket=library_next24_images" \
     --service-account=48099017975-compute@developer.gserviceaccount.com
```

Test the trigger
```shell
gsutil cp books/The_Jungle_Book-Rudyard_Kipling-1894-public.txt gs://books-${PROJECT_ID}
gsutil cp books/Meditations-Marcus_Aurelius-0161-public.txt gs://books-${PROJECT_ID}

gcloud logging read "resource.labels.service_name=books-genai-jit AND textPayload:GeekHour" --format=json
gcloud logging read "resource.labels.service_name=books-genai-jit AND textPayload:CloudRun" --format=json
```

Log capture:
```shell
 gcloud logging read "resource.labels.service_name=books-genai-jit AND textPayload:CloudRun" --format=json
 gcloud logging read "resource.labels.service_name=books-genai-native AND textPayload:CloudRun" --format=json
```

