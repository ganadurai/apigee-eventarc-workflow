# Apigee Event Processing via GCP Eventarc and Workflows

## Tracking and Processing Apigee Management Events 

Actions on Apigee management plane get recorded via audit logs to track events/changes that occur. Few to mention:

Actions of API Proxies (Create, Update, Deploy, etc)
Actions of API Products (Create, Update, Deploy, etc)
Actions on Developer Apps
and more.

For information about Apigee audit logs, see the details [here](https://cloud.google.com/apigee/docs/api-platform/debug/audit-logging#overview). 

To view a sample of these events in Cloud Logging for your GCP-Project / Apigee-Org execute the below (adjust the query based on your needs):

```bash
protoPayload.methodName=~"google.cloud.apigee.v1.Deployment*"
OR protoPayload.methodName=~"google.cloud.apigee.v1.Api*"
OR protoPayload.methodName=~"google.cloud.apigee.v1.Target*"
OR protoPayload.methodName=~"google.cloud.apigee.v1.Developer*"
OR protoPayload.methodName=~"google.cloud.apigee.v1.Environment*"
OR protoPayload.methodName=~"google.cloud.apigee.v1.Project*"
OR protoPayload.methodName=~"google.cloud.apigee.v1.Organization*"
NOT protoPayload.methodName="google.cloud.apigee.v1.RuntimeService.ReportInstanceStatus"
NOT protoPayload.methodName="google.cloud.apigee.v1.EnvironmentService.Subscribe"
NOT protoPayload.methodName="google.cloud.apigee.v1.EnvironmentService.Unsubscribe"
NOT protoPayload.methodName="google.cloud.apigee.v1.DeploymentService.GenerateDeployChangeReport"
```

There is always a need to better capture the above events and process the events by posting it to a Cloud Function, Pub/Sub, Cloud Run or o an external http endpoint.

One of the ways to achieve the above is by utilizing [GCP EventArc](https://cloud.google.com/eventarc/docs/overview). An Eventarc trigger enables to capture specific events from cloudlogging audit logs and act on it.  

## Sample Implementation

Follow the below steps to capture Apigee Developer create event via EventArc and post it to GCP Workflow. In this example the Workflow posts the audit log payload to an HTTP endpoint. Follow the steps within your GCP Cloudshell.

1. Initialize variables
    ```bash
    PROJECT_ID=<GCP Project Id>
    USER_ID=<GCP User email>
    TRIGGER_SA=<Service Account Name, created and used in this setup>

    gcloud config set project $PROJECT_ID
    export PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)')"
    ```

1. Enable the APIs
    ```bash
    gcloud services enable \
    logging.googleapis.com \
    eventarc.googleapis.com \
    workflows.googleapis.com \
    workflowexecutions.googleapis.com \
    pubsub.googleapis.com
    ```

1. Grant user to admin EventArc
    ```bash
    gcloud projects add-iam-policy-binding ${PROJECT_NUMBER} --member=user:$USER_ID --role=roles/eventarc.admin
    ```

1. Create Service Account and Assin the needed roles.
    ```bash
    gcloud iam service-accounts create ${TRIGGER_SA}

    gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member=serviceAccount:${TRIGGER_SA}@$PROJECT_ID.iam.gserviceaccount.com \
    --role=roles/workflows.invoker

    gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member "serviceAccount:${TRIGGER_SA}@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/eventarc.eventReceiver"

    gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member "serviceAccount:${TRIGGER_SA}@$PROJECT_ID.iam.gserviceaccount.com" \
    --role "roles/logging.logWriter"
    ```

1. Create Workflow yaml.
    ```bash
    cat <<EOF > workflow.yaml
    main:
        params: [input]
        steps:
        - registerPayload:
            call: http.post
            args:
                body:
                    payload: \${input}
                url: https://34.107.231.171.nip.io/apigee-hybrid-proxychain
            result: httpOutput
        - returnOutput:
                return: \${httpOutput.body}
    EOF
    ```

1. Deploy Workflow
    ```bash
    gcloud workflows deploy developer-create-trigger-workflow \
    --source=workflow.yaml --service-account=${TRIGGER_SA}@$PROJECT_ID.iam.gserviceaccount.com
    ```

1. Create Eventarc trigger that posts data to Workflow
    ```bash
    gcloud eventarc triggers create apigee-developer-create-workflows-trigger \
    --location=us-central1 \
    --destination-workflow=developer-create-trigger-workflow \
    --destination-workflow-location=us-central1 \
    --event-filters="type=google.cloud.audit.log.v1.written" \
    --event-filters="serviceName=apigee.googleapis.com" \
    --event-filters="methodName=google.cloud.apigee.v1.Developers.CreateDeveloper" \
    --service-account="${TRIGGER_SA}@${PROJECT_ID}.iam.gserviceaccount.com"
    ```

1. Validating the setup
  <ol type="a">
    <li>Create Developer via the Mgmt api or the console</li>
    <li>Check <a href="https://console.cloud.google.com/logs/query;query=protoPayload.@type%3D%22type.googleapis.com%2Fgoogle.cloud.audit.AuditLog%22%0AprotoPayload.methodName%3D%22google.cloud.apigee.v1.Developers.CreateDeveloper%22%0AprotoPayload.serviceName%3D%22apigee.googleapis.com%22;">Cloud logging</a> for the presence of the audit log in your sorresponding GCP project.</li>
    <li>Check <a href="https://console.cloud.google.com/eventarc/triggers/us-central1/apigee-developer-create-workflows-trigger">Eventarc</a> for the trigger invocations</li>
    <li>Check <a href="https://console.cloud.google.com/workflows/workflow/us-central1/developer-create-trigger-workflow/executions">Workflow</a> for the executions triggered via Eventarc.</li>
  </ol>
