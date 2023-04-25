

---
title: "Cloud Run Jobs"
date: 2022-07-10
draft: false
github_link: "https://medium.com/dev-genius/streamline-your-data-onboarding-with-dbt-codegen-package-8ecb5caab03c"
author: "Lior Kaufman"
tags:
  - Python
  - Cloud computing
  - Docker
  - ETL/ELT
# image: /images/post.jpg
description: "How-To dockerize and deploy a python script"
toc: 
---


Checking Out Google Cloud Run Jobs With a Practical Example
===========================================================

Deploy a Dockerized Python script to Google Cloud Platform using Cloud Run Jobs.
--------------------------------------------------------------------------------

In this article we will write a simple python data extraction script, deploy it to Google Cloud Run Jobs container and discuss the status of cloud run jobs — google newest container as a service offering.

**Prerequisites to follow this guide:**

*   Local installation of [Docker](https://docs.docker.com/get-docker/)
*   Google Cloud [CLI tool](https://cloud.google.com/sdk/docs/install)
*   Google Cloud account with billing enabled

Official overview of Cloud Run Jobs from Google
===============================================

> Although [Cloud Run](https://cloud.google.com/run) services are a good fit for containers that run indefinitely listening for HTTP requests, Cloud Run jobs may be a better fit for containers that run to completion and don’t serve requests. For example, processing records from a database, processing a list of files from a Cloud Storage bucket, or a long-running operation, such as calculating Pi, would work well if implemented as a Cloud Run job.
> 
> Jobs don’t have the ability to serve requests or listen on a port. This means that unlike Cloud Run services, jobs should not bundle a web server. Instead, jobs containers should exit when they are done.

Tools we are going to use:
==========================

*   GCloud CLI
*   Cloud Run Jobs
*   Cloud Container Registry
*   Google Cloud Storage (GCS)
*   Docker

The Script
==========

All the code can be found at [https://github.com/LiorKaufman/cloud-run-job-demo](https://github.com/LiorKaufman/cloud-run-job-demo)

We are going to write a short script that will call an external API, parse the response as a pandas dataframe and save it to GCS.  
Will be using [https://api.publicapis.org/](https://api.publicapis.org/random) a free API resource that catalogues other free public APIs.

**Make a directory and files**

```
mkdir cloud-run-jobs-demo  
cd cloud-run-jobs-demo   
touch main.py Dockerfile .dockerignore requirements.txt
```

**Setup virtual environment**

```
python3 -m venv venv  
source venv/bin/activate
```

**Install dependencies**

```
pip install requests google-cloud-storage pandas  
pip freeze > requirements.txt
```

Now that we have our local environment setup we can start writing some code. We need to call our API, parse the response and save it into GCS. Will be doing each step as a separate function and letting our main function run all steps.

**Calling the API**

```
import requests  
import pandas as pddef call_api():   
    url = "[https://api.publicapis.org/random](https://api.publicapis.org/random)"          
    resp = requests.get(url)  
    return resp.json()def main():  
    print("*** Starting extraction ***")   
    data = []  
    # Each call will give us a random entry from the api   
    for i in range(0,9):  
        entry = call_api()  
        data.append(entry["entries"][0])  
    df = pd.DataFrame(data)
```

**Uploading the data to GCS**

Before we can upload to GCS, we first need to create a new [bucket in GCP](https://cloud.google.com/storage/docs/creating-buckets#storage-create-bucket-python). In order to create a bucket we will need to have the correct IAM permissions. If you are unable to create a bucket you will have to contact you GCP administrator.

```
gsutil mb -p mygoogle_cloud_project_name gs://my_demo_bucket
```

If you are unsure if the bucket is created or want to use a different bucket you can list all available buckets using the following command.

```
gsutil ls -p PROJECT_ID
```

Now that we have a bucket we can upload our file to it. Let’s write a simple function that uploads the data we collect as a csv file.

```
def upload_dataframe_as_csv(df,bucket_name, destination_blob_name):  
    """Uploads a dataframe as csv file to the bucket."""  
    storage_client = storage.Client()  
    bucket = storage_client.bucket(bucket_name)  
    blob = bucket.blob(destination_blob_name)blob.upload_from_string(df.to_csv(index=False,header=True),"text/csv")print(  
        f"{destination_blob_name} with {df.shape[0]} rows got uploaded to bucket{bucket_name}."  
    )
```

Authentication
==============

In order to actually be able to upload or file into GCS we need to be authenticated to google. If you have the Gcloud CLI you can run the following command to authenticate using your own GCP user account.

```
gcloud auth login 
```

The other auth option is using a service account key. Will create the service account and assign it’s roles using the console instead of the CLI. Navigate to the service account page under IAM & Admin.

Click on button to create a service account fill in the name and email. Fill up the required fields and click continue and create.

Grant the Storage Object Admin rule to the new service account and click continue and done.

To generate the service account key:

*   Click the service account
*   Go to the Keys tab and click on the add key button
*   Choose key type of JSON
*   Save the file in a secure location on your machine

The google storage client automatically looks for service credentials in your local development environment. You can set the a new env variable with a path to the service_account_key.json file

```
export GOOGLE_APPLICATION_CREDENTIALS=<path-to-service-account-key>
```

Or alternatively you can run the following command to let the gcloud CLI tool deal with the auth using the service account.

```
gcloud auth activate-service-account --key-file <path-to-service-account-key>
```

**Testing our script**

Let's look at the full code we are about to test. We have our api_call function, our upload function and our main function that is coordinating all the steps.  
Additionally we should have the GOOGLE_APPLICATION_CREDENTIALS variable set.

```
import requests  
import pandas as pd  
from google.cloud import storage  
from os import environTASK_INDEX = environ.get("CLOUD_RUN_TASK_INDEX", 0)def call_api():   
    url = "[https://api.publicapis.org/random](https://api.publicapis.org/random)"          
    resp = requests.get(url)  
    return resp.json()def upload_dataframe_as_csv(df,bucket_name, destination_blob_name):  
    """Uploads a dataframe as csv file to the bucket."""  
    storage_client = storage.Client()  
    bucket = storage_client.bucket(bucket_name)  
    blob = bucket.blob(destination_blob_name)blob.upload_from_string(df.to_csv(index=False,header=True),"text/csv")print(  
        f"{destination_blob_name} with {df.shape[0]} rows got uploaded to bucket{bucket_name}."  
    )def main():  
    print("*** Starting extraction ***")   
    data = []  
    # Each call will give us a random entry from the api   
    for i in range(0,9):  
        entry = call_api()  
        data.append(entry["entries"][0])  
    df = pd.DataFrame(data)  
     upload_dataframe_as_csv(df,"my_demo_bucket_cloud_run",f"random_entires_{TASK_INDEX}.csv")  
if __name__ == '__main__':  
    main()
```

When we run our script, we should see the following message and in the bucket itself we should see our file.

Dockerizing our app and a final test
====================================

Our next step is to dockerize our script build a docker image and push it to google cloud container registry. Copy the following commands to your docker file.

```
FROM python:3.9-slimCOPY . .# INSTALL REQUIRED LIBRARIES  
RUN pip install -r requirements.txtCMD ["python", "-u", "main.py"]
```

Build the image locally

```
docker build . --tag gcr.io/PROJECT-ID/JOB-NAME:latest
```

To run the image successfully in our local environment we need to mount the service account key we created in the previous step. In order to successfully do that we first need to make sure we have the GOOGLE_APPLICATION_CREDENTIALS env variable is set

run the following command to set the env var and run the docker image

```
export GOOGLE_APPLICATION_CREDENTIALS=<path-to-file>docker run --rm \  
 -e GOOGLE_APPLICATION_CREDENTIALS=/tmp/keys/my_service_account.json \  
-v $GOOGLE_APPLICATION_CREDENTIALS:/tmp/keys/my_service_account.json:ro gcr.io/cloud-run-jobs-demo-354304/jobs-demo:latest
```

If everything is configured correctly you should see the same success message in the terminal .

The last step we need to do before pushing the image to the cloud is authenticating docker with gcp

```
gcloud auth configure-docker gcr.io
```

Now we just need to push our image.

```
docker push gcr.io/PROJECT-ID/JOB-NAME:latest
```

Creating and executing cloud run job
====================================

To create the cloud run job we just need to run the following command

```
gcloud beta run create jobs JOB-NAME --image gcr.io/PROJECT-ID/JOB-NAME:latest --region us-west4 --tasks 3
```

You should be able to see a new job created in the cloud run console

To execute a job we just run the following

```
gcloud beta run jobs execute JOB-NAME
```

A very nice feature of cloud run jobs is that they can also be triggered by an API endpoint. The following snippet will deal with the authentication and make an HTTP POST call triggering the job.

```
import requests  
from os import environ  
from google.oauth2 import service_account  
import google.auth  
import google.auth.transport.requestsSCOPES = [  
    "[https://www.googleapis.com/auth/cloud-platform](https://www.googleapis.com/auth/cloud-platform)",  
]SERVICE_ACCOUNT_FILE = environ.get("GOOGLE_APPLICATION_CREDENTIALS")  
credentials = service_account.Credentials.from_service_account_file(  
    SERVICE_ACCOUNT_FILE, scopes=SCOPES  
)  
auth_req = google.auth.transport.requests.Request()  
credentials.refresh(auth_req)headers = {  
"Authorization": "Bearer " + credentials.token,  
"Content-Type": "application/json",  
}  
url = "[https://us-west4-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/PROJECT-ID/jobs/JOB-NAME:run](https://us-central1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/sightly-dev-sandbox/jobs/ad-fontes-extract-job:run)"  
res = requests.post(url, headers=headers)
```

After triggering the job (choose whatever method works for your use case) cloud run jobs will log the output of the run and will also give a separate log per task executed.

Executed jobsOur storage bucket after running

Conclusions and notes
=====================

I hope this guide will prove useful if you are thinking of exploring cloud run jobs. Personally I find cloud run jobs to be an extremely convenient way to run dockerized scripts which don’t need an HTTP input. It looks to be ideal for batch processing of data, but I still want to add a few warning before deciding to use cloud run jobs.

1.  It’s still in beta meaning changes can still happen.
2.  New tool, less documentation and tutorials (hoping to change this).

3. No way to re run specific tasks (I haven’t seen any mention of this capability in the documentation )

4. My last point is more of an ask from Google.  
It’s possible to wait for task completion when triggering it using the CLI.

```
gcloud beta run jobs execute JOB_NAME --wait
```

This is great for complex workflows that require you to run multiple processes based on previous step completion. I did not find any mention of this capabilities for the job API endpoint . It’s a shame because now to use cloud run job in an orchestration tool (i.e Airflow) I would need a image with the the sdk image just to wait for the job completion vs a simple http to execute and wait.
