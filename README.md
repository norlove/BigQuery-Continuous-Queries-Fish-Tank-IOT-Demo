# BigQuery-Continuous-Queries-Fish-Tank-IOT-Demo
This is an end-to-end demo using BigQuery continuous queries to address an unhealthy fish tank IOT sensor event with ServiceNow's Service Operations Workspace. This was demoed onstage at Google Cloud Next 2025 [recording](https://www.youtube.com/watch?v=SKv6bbwSL5I).

BigQuery continuous queries are SQL statements that run continuously, allowing you analyze incoming data in BigQuery in real time. You can insert the output rows produced by a continuous query into a BigQuery table or export them to Pub/Sub, Bigtable, or Spanner.

Documentation on BigQuery continuous queries can be found [HERE](https://cloud.google.com/bigquery/docs/continuous-queries-introduction), and a blog which provides context of BigQuery continuous queries can be found [HERE](https://cloud.google.com/blog/products/data-analytics/bigquery-continuous-queries-makes-data-analysis-real-time).

----------------------------------------------------------------------------------------------------
Let's assume our pet store has IOT sensors on each of our fish tanks to ensure our fish are happy and healthy with safe water conditions. 
It accomplishes this by streaming IOT sensor data to a table in BigQuery by way of a Pub/Sub topic and Pub/Sub to BigQuery subscription.
From here, a continuous query is running which processes the incoming IOT sensor data in real time, checks if the water conditions are unhealthy, and if unhealthy calls out to a Gemini 2.0 flash model to determine the possible cause of the issue and a quick remedy. 
The continuous query then writes this message to another Pub/Sub topic, which is received by ServiceNow’s Workflow Data Fabric that creates a service ticket for the pet store team to immediately rectify the problem with the fish tank.

<img width="970" alt="Screenshot 2025-04-22 at 9 57 56 PM" src="https://github.com/user-attachments/assets/f49f6c7e-94b5-42e2-abc1-5828c0c7a7f5" />

## Setting up your project

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

4. Create a BigQuery remote connection named "continuous-queries-connection" in the Cloud Console using [these steps](https://cloud.google.com/bigquery/docs/bigquery-ml-remote-model-tutorial#Create-Connection).

    <img width="544" alt="Screenshot 2024-07-28 at 3 46 08 PM" src="https://github.com/user-attachments/assets/05afada9-a7aa-4cbf-80d6-c075f0a23d4d">

5. After the connection has been created, click "Go to connection", and in the Connection Info pane, copy the service account ID for use in the next step.

6. Grant Vertex AI User role IAM access to the service account ID you just copied using [these steps](https://cloud.google.com/bigquery/docs/bigquery-ml-remote-model-tutorial#set_up_connection_access).

7. Create a BigQuery ML remote model with Gemini 2.0 Flash by running the following SQL query in your BigQuery environment:
      ```
      #Creates a BigQuery ML remote model named gemini_2_0_flash
      CREATE OR REPLACE MODEL `Cymbal_Pets.gemini_2_0_flash`
      REMOTE WITH CONNECTION `us.continuous-queries-connection`
      OPTIONS (ENDPOINT = 'gemini-2.0-flash');
      ```

9. Create a BigQuery Service Account named "bq-continuous-query-sa", granting yourself permissions to submit a job that runs using the service account [[ref](https://cloud.google.com/bigquery/docs/continuous-queries#user_account_permissions)], and granting permissions to the service account itself to access BigQuery resources [[ref](https://cloud.google.com/bigquery/docs/continuous-queries#service_account_permissions)].

**NOTE: if you have issues with this demo, it is 9 times out of 10 related to an IAM permissions issue.**

## Setup the Pub/Sub topics and subscriptions

1. Create a Pub/Sub topic [[ref](https://cloud.google.com/pubsub/docs/create-topic)] named "cymbal_pets_BQ_writer", which will write data into BigQuery. No need to create a default subscription.

     ![Screenshot 2025-04-24 at 10 53 10 AM](https://github.com/user-attachments/assets/e05b711a-f5c0-4c72-8b3d-b820171b9fcd)

3. Create another Pub/Sub topic named "cymbal_pets_ServiceNow_writer", which will write data from BigQuery into ServiceNow. Again, no need to create a default subscription.
   
4. Grant the service account you created in step #8 above permissions to the Pub/Sub topic with the Pub/Sub Viewer and Pub/Sub Publisher roles [[ref](https://cloud.google.com/bigquery/docs/export-to-pubsub#service_account_permissions_2)].

6. Create a Pub/Sub subscription named "cymbal_pets_to_BigQuery_table" under the Pub/Sub topic cymbal_pets_BQ_writer which will be a BigQuery to Pub/Sub subscription and will dump data into the BQ table previously created. Be sure to use the BQ table schema.

    ![Screenshot 2025-04-24 at 1 24 31 PM](https://github.com/user-attachments/assets/649ed089-7b18-4cf4-8453-4d1e998ff84f)

7. Grant your project's internal Google-provided Pub/Sub service account BigQuery Data Editor and BigQuery Metadtaa Viewer permissions [[ref](https://cloud.google.com/pubsub/docs/create-bigquery-subscription#assign_bigquery_service_account)]. This is to allow the Pub/Sub subscription cymbal_pets_to_BigQuery_table to write data into your BigQuery table. To do this, you'll grant the service account service-{PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com (replacing PROJECT_NUMBER with your project's actual project number) with BigQuery Data Editor and BigQuery Metadata Viewer role permissions.

    ![Screenshot 2025-04-24 at 1 17 45 PM](https://github.com/user-attachments/assets/e5d834b4-1fc5-499b-8eda-f5292bb481ec)

8. Test the Pub/Sub to BigQuery subscription by clicking on the cymbal_pets_BQ_writer Pub/Sub topic, go to the Messages tab, and click publish message. Provide this input in the message body:

    ```
    {
    "event_timestamp": "2025-03-18T22:24:57.206020Z",
    "store_id": "store-789",
    "location": {
      "city": "Springfield",
      "state": "IL",
      "zip_code": "62704"
    },
    "tank_id": "fish-101",
    "temperature": 56.2,
    "ph_level": 7.8,
    "ammonia_ppm": 0.25,
    "nitrite_no2_ppm": 0.1,
    "nitrate_no3_ppm": 30.0,
    "salinity_ppm": 35.0,
    "event_metadata": {}
    }
    ```
    
9. Go to your BigQuery table and query it to make sure the one record was successfully inserted.
        ![Screenshot 2025-04-24 at 1 31 06 PM](https://github.com/user-attachments/assets/fbdc475e-7fa8-4906-ac44-5858d2f7b5e5)


10. Now delete that record to make sure the table is clean:
    ```DELETE `Cymbal_Pets.Fish_Tank_IOT_Data_Ingest` WHERE TRUE```

## Configure ServiceNow (if applicable)

1. INCLUDE STEPS TO CONFIGURE SERVICENOW ENDPOINT HERE

2. Once your ServiceNow connection is configured and you have the endpoint configured, create a new Pub/Sub subcription named "cymbal_pets_to_servicenow" under the Pub/Sub topic cymbal_pets_ServiceNow_writer. This Pub/Sub subscription will push messages from the continuous query into the ServiceNow workflow. Be sure to set up the Pub/Sub scription as a Push method with the endpoint URL you created above. Also be sure to check the "Enable payload unwrapping" and "Write metadata fields".

    ![B9JUuv7YZ2ATCVL](https://github.com/user-attachments/assets/a607e475-eb5a-45ef-a2b2-0817d401022e)


3. Test the Pub/Sub to ServiceNow subscription by clicking on the cymbal_pets_ServiceNow_writer Pub/Sub topic, go to the Messages tab, and click publish message. Provide this input in the message body:

    ```
    {
    "event_timestamp": "2025-03-18T22:24:57.206020Z",
    "ticket_description": "Just a test message to ServiceNow"
    }
    ```

4. Log into ServiceNow's Service Operations Workspace, go to your inbox, make sure your status is set as "avaialble" and confirm a new incident arrived.

    ![4sNxqG4YTYfDJhF](https://github.com/user-attachments/assets/8330c9fa-4c80-4b1d-9c42-b5159ccf75ff)

5. Accept the incident and make sure the ticket description was correctly carried over.

   ![BXsc7h5gDxzMjwi](https://github.com/user-attachments/assets/f5922f2f-0294-42f6-9a1f-752afcc42385)


## Create a BigQuery continuous query

1. BigQuery continuous queries require a BigQuery Enterprise or Enterprise Plus reservation [[ref](https://cloud.google.com/bigquery/docs/continuous-queries-introduction#reservation_limitations)]. Create one now named "bq-continuous-queries-reservation" in the US multi-region, with a max reservation size of 50 slots, and a slot baseline of 0 slots (to leverage slot autoscaling).
   
      ![Screenshot 2025-04-24 at 1 50 22 PM](https://github.com/user-attachments/assets/36ae54de-05fb-4cb7-a88f-a73ff92c73b4)

3. Once the reservation has been created, click on the three dots under Actions, and click "Create assignment". 

      <img width="212" alt="Screenshot 2024-08-01 at 6 26 21 PM" src="https://github.com/user-attachments/assets/2d71fe08-d3c0-4d35-ab4a-769120f535e4">

4. Click Browse and find the project you are using for this demo. Then Select "CONTINUOUS" as the Job Type. Click Create.

        ![Screenshot 2025-04-24 at 1 51 55 PM](https://github.com/user-attachments/assets/da7cdd87-00b3-45fe-b501-f6a45e3a1343)

6. You'll now see your assignment created under your reservation:
   
      ![Screenshot 2025-04-24 at 1 54 23 PM](https://github.com/user-attachments/assets/dbf89e4d-82b9-4292-906f-3651c6de6278)


7. Go back to the BigQuery SQL editor and paste the following SQL query:

   **Note: The URI provided in the example below specifies a Pub/Sub Topic as the destination for the continuous query, with the GCP project listed as "my_project" and the Pub/Sub Topic listed as "my_topic". Be sure to change these to your own project/topic. You can also specify different destinations for a BigQuery continuous query, as described in [these examples](https://cloud.google.com/bigquery/docs/continuous-queries#examples).**

   ```

   ```

## Generate synthetic IoT sensor data 

1. XYZ

## Test everything end-to-end

1. XYZ

