# Cloud Run and Cloud logging
In this repository, we'll test the integration of Cloud Run and Cloud Logging to create a "retry &amp; alert" logic.

First, we craft a dummy Cloud Run service to test your logging, retry, and alerting logic. To test the dummy service locally, run the following commands:

```bash
cd docker
```
```bash
docker build -t dummy-service .
```
```bash
docker run -d -p 8080:8080 --name dummy-container dummy-service
```
```bash
python3 test.py
```

Run the last line a few times, you will get the responses `{'message': 'Successfully processed SKU: 1234'}` and `{'error': 'Failed to process SKU: 1234'}` with a 70-30 probability (you can modify this in [`main.py`](docker/main.py)).

#### Deploy Dummy Cloud Run 
First, set your project environment variables: 
```bash
export PROJECT_ID='your-project-id'
export NAME_SERVICE='cloud-run-service-name'
export LOCATION='location service'
```
`LOCATION` is the region where your Artifact Registry repository is located.

Now, make sure to authenticate with your project credentials and that your Docker client is authenticated, too.
```bash
gcloud auth login
```
```bash
gcloud auth configure-docker $LOCATION-docker.pkg.dev
```

Lastly, deploy the Cloud Run service using Cloud Build:

```bash
gcloud builds submit --tag gcr.io/$PROJECT_ID/dummy-service
```
```bash
gcloud run deploy --image gcr.io/$PROJECT_ID/dummy-service --platform managed --allow-unauthenticated 
```

## Test Logging!

Let's break down how we can achieve robust logging, retry logic, and alerting for Cloud Run async calls.

* **Cloud Run Service:** existing Cloud Run service, processing SKU-related tasks.
* **Python Client:** Python application initiates async calls to the Cloud Run service and logs each call attempt with SKU, status (success/failure), attempt number, and other relevant metadata using Cloud Logging's structured format. See the file [`client.py`](client.py).  

    To create success/failure logs, first, create an environment and set your environment variables:
    ```bash
    python3 -m venv cloudrun-env
    source cloudrun-env/bin/activate
    pip3 install -r requirements.txt
    ```
    ```bash
    export CLOUD_RUN_SERVICE_URL='your-cloud-run-service-url'
    export PROJECT_ID='your-project-id'
    ```
    Authenticate with your credentials (or using a Service Account):
    ```bash
    gcloud auth application-default login
    ```
    Run client test:
    ```bash
    python3 client.py
    ```

* **Log-based Metrics:** Create a log-based metric in Cloud Monitoring to filter for failed call logs.  

    ![log-based](images/log-based%20metric.png)
    For *name* you can use `cloud_run_failed_all_retries` and for *filter* try:
    ```bash
    resource.type="global"
    logName="projects/YOUR-PROJECT-ID/logs/cloud_run_async_calls"
    jsonPayload.status="failed_all_retries"
    ```
    This is to create an alert only after we've gone over the maximum attempts for an SKU without getting a response. 

* **Alerting Policy:** Set up an alerting policy in Cloud Monitoring based on the log-based metric. This policy will trigger after a defined threshold of failed attempts for a specific SKU.
