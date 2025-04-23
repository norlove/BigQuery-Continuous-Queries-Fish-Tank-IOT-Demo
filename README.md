# BigQuery-Continuous-Queries-Fish-Tank-Health-IOT-Demo
This is an end-to-end demo using BigQuery continuous queries to address an unhealthy fish tank IOT sensor event with ServiceNow's Service Operations Workspace. This was demoed onstage at Google Cloud Next 2025 [recording](https://www.youtube.com/watch?v=SKv6bbwSL5I).

BigQuery continuous queries are SQL statements that run continuously, allowing you analyze incoming data in BigQuery in real time. You can insert the output rows produced by a continuous query into a BigQuery table or export them to Pub/Sub, Bigtable, or Spanner.

Documentation on BigQuery continuous queries can be found [HERE](https://cloud.google.com/bigquery/docs/continuous-queries-introduction), and a blog which provides context of BigQuery continuous queries can be found [HERE](https://cloud.google.com/blog/products/data-analytics/bigquery-continuous-queries-makes-data-analysis-real-time).

----------------------------------------------------------------------------------------------------
Let's assume our pet store has IOT sensors on each of our fish tanks to ensure our fish are happy and healthy with safe water conditions. 
It accomplishes this by streaming IOT sensor data to a table in BigQuery by way of a Pub/Sub topic and Pub/Sub to BigQuery subscription.
From here, a continuous query is running which processes the incoming IOT sensor data in real time, checks if the water conditions are unhealthy, and if unhealthy calls out to a Gemini 2.0 flash model to determine the possible cause of the issue and a quick remedy. 
The continuous query then writes this message to another Pub/Sub topic, which is received by ServiceNow’s Workflow Data Fabric that creates a service ticket for the pet store team to immediately rectify the problem with the fish tank.

<img width="970" alt="Screenshot 2025-04-22 at 9 57 56 PM" src="https://github.com/user-attachments/assets/f49f6c7e-94b5-42e2-abc1-5828c0c7a7f5" />

## Setting up our project

1. Ensure your project has enabled the [Vertex AI API](https://console.cloud.google.com/apis/library/aiplatform.googleapis.com) and [Pub/Sub API](https://console.cloud.google.com/apis/library/pubsub.googleapis.com)

2. Ensure your user account has the appropriate IAM permissions [[ref](https://cloud.google.com/bigquery/docs/continuous-queries#choose_an_account_type)]. During this demo, we'll run the continuous query with a Service Account as we'll be writing to a Pub/Sub topic.

3. Create a dataset and table in your project by running the following SQL query in your BigQuery environment:
```
#Creates a dataset named Cymbal_Pets within your currently selected project
CREATE SCHEMA IF NOT EXISTS Cymbal_Pets
OPTIONS (
  location = "US"
);

#Creates a table named Fish_Tank_IOT_Data_Ingest.
CREATE OR REPLACE TABLE
  `Cymbal_Pets.Fish_Tank_IOT_Data_Ingest` ( 
    event_timestamp TIMESTAMP,
    store_id STRING,
    location STRUCT<
      city STRING,
      state STRING,
      zip_code STRING>,
    tank_id STRING,
    temperature FLOAT64,
    ph_level FLOAT64,
    ammonia_ppm FLOAT64,
    nitrite_no2_ppm FLOAT64,
    nitrate_no3_ppm FLOAT64,
    salinity_ppm FLOAT64,
    event_metadata JSON )
CLUSTER BY
  event_timestamp;
```

4. Create a BigQuery remote connection named "continuous-queries-connection" in the Cloud Console using [these steps](https://cloud.google.com/bigquery/docs/bigquery-ml-remote-model-tutorial#create_a_cloud_resource_connection).

<img width="544" alt="Screenshot 2024-07-28 at 3 46 08 PM" src="https://github.com/user-attachments/assets/05afada9-a7aa-4cbf-80d6-c075f0a23d4d">

5. After the connection has been created, click "Go to connection", and in the Connection Info pane, copy the service account ID for use in the next step.

6. Grant Vertex AI User role IAM access to the service account ID you just copied using [these steps](https://cloud.google.com/bigquery/docs/bigquery-ml-remote-model-tutorial#set_up_access).

7. Create a BigQuery ML remote model with Gemini 2.0 Flash by running the following SQL query in your BigQuery environment:
      ```
      #Creates a BigQuery ML remote model named gemini_2_0_flash
      CREATE OR REPLACE MODEL `Cymbal_Pets.gemini_2_0_flash`
      REMOTE WITH CONNECTION `us.continuous-queries-connection`
      OPTIONS (ENDPOINT = 'gemini-2.0-flash');
      ```
8. Create a BigQuery Service Account named "bq-continuous-query-sa", granting yourself permissions to subit a job that runs using the service account [[ref](https://cloud.google.com/bigquery/docs/continuous-queries#user_account_permissions)], and granting permissions to the service account itself to access BigQuery resources [[ref](https://cloud.google.com/bigquery/docs/continuous-queries#service_account_permissions)].

**NOTE: if you have issues with this demo, it is 9 times out of 10 related to an IAM permissions issue.**

## Setup the Pub/Sub topics and subscriptions

1. XYZ

## Configure ServiceNow (if applicable)

1. XYZ

## Create a BigQuery continuous query

1. XYZ

## Generate synthetic IoT sensor data 

1. XYZ

## Test everything end-to-end

1. XYZ

