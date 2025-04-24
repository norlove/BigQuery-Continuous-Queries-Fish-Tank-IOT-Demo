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

1. Ensure your project has enabled the [Compute Engine API](https://console.cloud.google.com/apis/library/compute.googleapis.com), [Vertex AI API](https://console.cloud.google.com/apis/library/aiplatform.googleapis.com) and [Pub/Sub API](https://console.cloud.google.com/apis/library/pubsub.googleapis.com)

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

9. Create a BigQuery Service Account named "bq-continuous-query-sa", granting yourself permissions to submit a job that runs using the service account [[ref](https://cloud.google.com/bigquery/docs/continuous-queries#user_account_permissions)], and granting permissions to the service account itself to access BigQuery resources [[ref](https://cloud.google.com/bigquery/docs/continuous-queries#service_account_permissions)] and writing to Pub/Sub.
    The to easily permission this, grant the following IAM roles to this service account:
   
    BigQuery Data Editor, BigQuery Data Viewer, BigQuery Connection User, Pub/Sub Publisher, and Pub/Sub Viewer

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

1. **TO DO: INCLUDE STEPS TO CONFIGURE SERVICENOW ENDPOINT HERE**

2. Once your ServiceNow connection is configured and you have the endpoint configured, create a new Pub/Sub subcription named "cymbal_pets_to_servicenow" under the Pub/Sub topic cymbal_pets_ServiceNow_writer. This Pub/Sub subscription will push messages from the continuous query into the ServiceNow workflow. Be sure to set up the Pub/Sub scription as a Push method with the endpoint URL you created above (For Googler's please reference the instructions [HERE](https://docs.google.com/document/d/1Z6ZUwhSOPSsmPLsvEUZZCG9ZBux-U3kfKQRxTNsmBYE/edit?usp=sharing&resourcekey=0-t1GPrH6S_UGPqkw2DZu5iA)). Also be sure to check the "Enable payload unwrapping" and "Write metadata fields".

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


7. Go back to the BigQuery SQL editor and paste the below SQL query. Save your query.

   **Note: The URI provided in the example below specifies a Pub/Sub Topic as the destination for the continuous query, with the GCP project listed as "my_project". Be sure to change these to your own project. You can also specify different destinations for a BigQuery continuous query, as described in [these examples](https://cloud.google.com/bigquery/docs/continuous-queries#examples).**

   ```
    EXPORT DATA
    OPTIONS (format = CLOUD_PUBSUB,
    uri = "https://pubsub.googleapis.com/projects/my_project/topics/cymbal_pets_ServiceNow_writer")
    AS (SELECT
     TO_JSON_STRING(
       STRUCT(
         event_timestamp,
         store_id,
         location,
         tank_id,
         temperature,
         ph_level,
         ammonia_ppm,
         nitrite_no2_ppm,
         nitrate_no3_ppm,
         salinity_ppm,
         ml_generate_text_llm_result as ticket_description))
    FROM ML.GENERATE_TEXT( MODEL `Cymbal_Pets.gemini_2_0_flash`,
       (SELECT
         event_timestamp,
         store_id,
         location,
         tank_id,
         temperature,
         ph_level,
         ammonia_ppm,
         nitrite_no2_ppm,
         nitrate_no3_ppm,
         salinity_ppm,
         CONCAT("Analyze the following salt water fish tank IOT sensor readings and create a support ticket problem description. ",
         "Readings -- ",
             "pH: ", ph_level, ", ",
             "Ammonia (NH3): ", ammonia_ppm, " ppm, ",
             "Nitrite (NO2): ", nitrite_no2_ppm, " ppm, ",
             "Nitrate (NO3): ", nitrate_no3_ppm, " ppm, ",
             "Temperature: ", temperature, " °F, ",
             "Salinity: ", salinity_ppm, " ppm. ",
         "Based on these values, what are the most likely causes for these readings, and what are the immediate recommended actions to ensure the safety and health of the aquarium inhabitants? Be specific about steps like water changes, additive usage, or equipment checks. Please keep the output to 250 words or less.") AS prompt
        FROM
            APPENDS(TABLE `Cymbal_Pets.Fish_Tank_IOT_Data_Ingest`,
              CURRENT_TIMESTAMP() - INTERVAL 10 MINUTE)
         WHERE(
           #Check for ranges outside of normal
           ph_level NOT BETWEEN 8.0 and 8.5
           OR ammonia_ppm > 0.05
           OR nitrite_no2_ppm > 0.1
           OR nitrate_no3_ppm > 50
           OR temperature NOT BETWEEN 74.0 AND 80.0
           OR salinity_ppm NOT BETWEEN 31.0 AND 36.0
         )),
       STRUCT( 1024 AS max_output_tokens,
       0.2 AS temperature,
       1 AS candidate_count,
       TRUE AS flatten_json_output)))
   ```
   
8.  Before you can run your query, you must enable BigQuery continuous query mode. In the BigQuery editor, click More -> Continuous Query mode

      ![Screenshot 2025-04-24 at 2 04 31 PM](https://github.com/user-attachments/assets/efbc8d44-aea6-4ac8-b64b-4008c6e195f7)


9. When the window opens, click the button CONFIRM to enable continuous queries for this BigQuery editor tab.

10. Since we are writing the results of this continuous query to a Pub/Sub topic, you must run this query using a Service Account [[ref](https://cloud.google.com/bigquery/docs/continuous-queries#choose_an_account_type)]. We'll use the service account we created earlier. Click More -> Query Settings and scroll down to the Continuous query section and select your service account "bq-continuous-query-sa" and click Save.

      <img width="551" alt="Screenshot 2024-07-29 at 12 03 55 AM" src="https://github.com/user-attachments/assets/28aff716-a3b9-4c85-a829-33efed32cd03">

11. Your continuous query should now be valid.

      ![Screenshot 2025-04-24 at 2 08 31 PM](https://github.com/user-attachments/assets/bebd7915-694d-4ba3-99b0-1d2fb25827e2)

12. Click Run to start your continuous query. After about a minute or so, the continuous query will be fully running, ready to receive and process incoming data into your abandoned_carts table.

## Generate synthetic IoT sensor data 

1. To demonstrate this environment at scale, let's generate some "healthy" fish tank IOT sensor data. We're going to do this using a Colab Enterprise notebook in BigQuery [[ref](https://cloud.google.com/bigquery/docs/notebooks-introduction)].

2. Go into BigQuery, click Create new Notebook. If required click the button to enable the API (there may be other APIs the wizard will have you enable).

    ![Screenshot 2025-04-24 at 2 14 32 PM](https://github.com/user-attachments/assets/8916fe51-7621-4e9f-830f-c5bfa47be719)


3. Click to add a new Code block. Paste in the following Python code. Be sure to change the project ID below from "my_project" to your own project.
   Save your notebook.

    ```
    # @title Generate healthy synthetic fish tank data and stream to Pub/Sub
    
    import time
    import datetime
    import random
    import json
    import logging
    import os
    from google.cloud import pubsub_v1
    from concurrent.futures import Future
    import google.auth
    
    # Project ID and Pub/Sub topic configuration
    PROJECT_ID = "my_project"
    TOPIC_ID = "cymbal_pets_BQ_writer"
    
    
    # Target Pub/Sub publishing rate and reporting interval
    MESSAGES_PER_SECOND_TARGET = 1000
    REPORTING_INTERVAL_SECONDS = 10
    
    # Configure logging for standard output
    logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
    
    # Generate synthetic data Cymbal Pet stores across a variety of cities
    US_LOCATIONS = [
        {"city": "Los Angeles", "state": "CA", "zip_code": "90012"},
        {"city": "San Francisco", "state": "CA", "zip_code": "94103"},
        {"city": "Las Vegas", "state": "NV", "zip_code": "88901"},
        {"city": "Seattle", "state": "WA", "zip_code": "98104"},
        {"city": "Portland", "state": "OR", "zip_code": "97204"},
        {"city": "Denver", "state": "CO", "zip_code": "80202"},
        {"city": "Phoenix", "state": "AZ", "zip_code": "85003"},
        {"city": "Salt Lake City", "state": "UT", "zip_code": "84111"},
        {"city": "Chicago", "state": "IL", "zip_code": "60602"},
        {"city": "Detroit", "state": "MI", "zip_code": "48226"},
        {"city": "Minneapolis", "state": "MN", "zip_code": "55415"},
        {"city": "Kansas City", "state": "MO", "zip_code": "64106"},
        {"city": "Atlanta", "state": "GA", "zip_code": "30303"},
        {"city": "Dallas", "state": "TX", "zip_code": "75201"},
        {"city": "Houston", "state": "TX", "zip_code": "77002"},
        {"city": "Miami", "state": "FL", "zip_code": "33132"},
        {"city": "Nashville", "state": "TN", "zip_code": "37219"},
        {"city": "New York", "state": "NY", "zip_code": "10007"},
        {"city": "Boston", "state": "MA", "zip_code": "02108"},
        {"city": "Philadelphia", "state": "PA", "zip_code": "19107"},
    ]
    
    # --- Healthy Ranges for Saltwater Aquarium ---
    HEALTHY_RANGES = {
        "ph_level": (8.1, 8.4),
        "ammonia_ppm": (0.0, 0.02),
        "nitrite_no2_ppm": (0.0, 0.05),
        "nitrate_no3_ppm": (2.0, 8.0),
        "temperature": (75.0, 79.0),
        "salinity_ppm": (32.0, 35.0)
    }
    
    # Global counters for stats
    messages_published_count = 0
    messages_failed_count = 0
    
    # --- Data Generation Function ---
    def generate_fish_tank_event() -> dict:
        """
        Generates a single synthetic HEALTHY fish tank sensor event for a store.
        Metadata is converted to a JSON string.
        """
        # Get current UTC time
        now_utc = datetime.datetime.now(datetime.timezone.utc)
        event_timestamp = now_utc.strftime('%Y-%m-%d %H:%M:%S.%f') + " UTC"
    
    
        # Generate a random store and fish tank ID
        store_id = f"store-{random.randrange(1, 101):03}"
        # Up to 15 fish tanks per store
        tank_id = f"tank-{random.randrange(1, 15):02}"
    
        # Generate healthy fish tank sensor readings (Using original keys)
        ph = random.uniform(*HEALTHY_RANGES["ph_level"])
        ammonia = random.uniform(*HEALTHY_RANGES["ammonia_ppm"])
        nitrite = random.uniform(*HEALTHY_RANGES["nitrite_no2_ppm"])
        nitrate = random.uniform(*HEALTHY_RANGES["nitrate_no3_ppm"])
        temperature = random.uniform(*HEALTHY_RANGES["temperature"])
        salinity = random.uniform(*HEALTHY_RANGES["salinity_ppm"])
    
        # Random location from the list
        location_data = random.choice(US_LOCATIONS)
    
        # Create metadata specific to fish tank
        metadata_dict = {
            "water_type": "saltwater",
            "sensor_set_id": f"tank-sensors-{store_id}-{tank_id}",
            "IOT_Sensor_status": "online"
        }
    
        # Convert the metadata dictionary into a JSON formatted string (kept pattern)
        event_metadata_string = json.dumps(metadata_dict)
    
        # Build the final event dictionary with fish tank fields
        event = {
            # --- Assign the RANDOMLY selected timestamp value ---
            "event_timestamp": event_timestamp,
            "store_id": store_id,
            "tank_id": tank_id,
            "ph_level": round(ph, 2),
            "ammonia_ppm": round(ammonia, 3),
            "nitrite_no2_ppm": round(nitrite, 3),
            "nitrate_no3_ppm": round(nitrate, 1),
            "temperature": round(temperature, 1),
            "salinity_ppm": round(salinity, 1),
            "location": {
                "city": location_data["city"],
                "state": location_data["state"],
                "zip_code": location_data["zip_code"]
            },
            "event_metadata": event_metadata_string
        }
        return event
    
    def publish_callback(future: Future):
        """Callback function to handle asynchronous publish results."""
        global messages_published_count, messages_failed_count
        try:
            message_id = future.result(timeout=60)
            messages_published_count += 1
        except TimeoutError:
            messages_failed_count += 1
            logging.warning("Publish timed out.")
        except Exception as e:
            messages_failed_count += 1
            logging.error(f"Failed to publish message: {e}", exc_info=False)
    
    # --- Publishing Function ---
    def publish_synthetic_data(project_id: str, topic_id: str, rate: int):
        """Generates and publishes synthetic fish tank data to Pub/Sub at a target rate."""
        global messages_published_count, messages_failed_count
    
        # Configure batch settings
        batch_settings = pubsub_v1.types.BatchSettings(
            max_messages=1000,
            max_bytes=1024 * 1024 * 5,
            max_latency=0.1,
        )
    
        try:
            # Create the Publisher client
            publisher = pubsub_v1.PublisherClient(batch_settings=batch_settings)
            topic_path = publisher.topic_path(project_id, topic_id)
    
            # --- Logging ---
            logging.info(f"--- Starting Fish Tank Data Publisher ---")
            logging.info(f"Publishing to Pub/Sub topic: {topic_path}")
            logging.info(f"Target publish rate: {rate} messages/second")
            logging.info(f"Generating HEALTHY fish tank data ranges.")
            logging.info(f"Event metadata sent as JSON string.")
            logging.info(">>> IMPORTANT: To stop, use Colab 'Interrupt execution' button <<<")
    
        except Exception as e:
            logging.error(f"FATAL: Failed to initialize PublisherClient or topic path: {e}", exc_info=True)
            return
    
        last_report_time = time.time()
        start_time_global = time.time()
    
        try:
            # Main publishing loop
            while True:
                batch_start_time = time.time()
                futures = []
    
                for _ in range(rate):
                    # --- Use the modified data generation function ---
                    event_data = generate_fish_tank_event()
                    data_bytes = json.dumps(event_data).encode("utf-8")
                    future = publisher.publish(topic_path, data=data_bytes)
                    future.add_done_callback(publish_callback)
                    futures.append(future)
    
                # Rate limiting logic
                batch_end_time = time.time()
                elapsed_time = batch_end_time - batch_start_time
                sleep_time = 1.0 - elapsed_time
                if sleep_time > 0:
                    time.sleep(sleep_time)
    
                # Reporting logic
                current_time = time.time()
                if current_time - last_report_time >= REPORTING_INTERVAL_SECONDS:
                    elapsed_total = current_time - start_time_global
                    current_total_published = messages_published_count
                    current_total_failed = messages_failed_count
                    avg_rate = current_total_published / elapsed_total if elapsed_total > 0 else 0
    
                    logging.info(
                        f"Published: {current_total_published}, Failed: {current_total_failed}, "
                        f"Avg Rate: {avg_rate:.2f} msg/sec"
                    )
                    last_report_time = current_time
    
        except KeyboardInterrupt:
            logging.info("Shutdown signal received. Stopping publisher...")
        except Exception as e:
            logging.error(f"An unexpected error occurred in the main loop: {e}", exc_info=True)
        finally:
            # Final stats logic
            elapsed_total = time.time() - start_time_global
            avg_rate = messages_published_count / elapsed_total if elapsed_total > 0 else 0
            logging.info("--------------------")
            logging.info("Final Run Statistics:")
            logging.info(f"Total Time Elapsed: {elapsed_total:.2f} seconds")
            logging.info(f"Total Messages Successfully Published: {messages_published_count}")
            logging.info(f"Total Messages Failed to Publish: {messages_failed_count}")
            logging.info(f"Overall Average Publish Rate: {avg_rate:.2f} msg/sec")
            logging.info("Publisher finished.")
    
    
    # --- Main Execution Block ---
    if __name__ == "__main__":
        # Keeping original checks, adjust TOPIC_ID placeholder if needed for your logic
        if TOPIC_ID == "your-fish-tank-topic-id" or not TOPIC_ID:
            logging.error("FATAL: Please update the TOPIC_ID variable for the fish tank data before running.")
        elif not PROJECT_ID or PROJECT_ID == "your-gcp-project-id":
             logging.error("FATAL: PROJECT_ID is not set correctly. Check auto-detection or set manually.")
        else:
            # Start the data publishing process
            publish_synthetic_data(PROJECT_ID, TOPIC_ID, MESSAGES_PER_SECOND_TARGET)
    
    ```

5. Click to connect the runtime. Give it a minute to activate and connect, represented by a green check mark.

    ![Screenshot 2025-04-24 at 2 28 57 PM](https://github.com/user-attachments/assets/f81bfcd3-a880-4029-86b3-8e0d21b05854)

6. Run the Notebook code cell.

   ![AmVzYi8fd4CbtDb](https://github.com/user-attachments/assets/f90f14a5-e68e-4af7-8124-b15aee5d4703)


7. The code will start and will start writing data into the Pub/Sub topic. This notebook will run until cancelled or the runtime expires (generally in a matter of hours).

8. Query your Fish_Tank_IOT_Data_Ingest BigQuery table again and observe that it is now receiving streaming data.

   ![Screenshot 2025-04-24 at 2 32 42 PM](https://github.com/user-attachments/assets/f06b768d-9487-46bd-bde3-af8699e45148)


## Insert an "unhealthy" fish tank IOT sensor event and confirm its receipt in ServiceNow

1. All the synthetic data being generated by the Python notebook is "healthy" fish tank water IOT events. Thus the continuous query isn't having to process any of the incoming data. Therefore, let's insert an "unhealthy" event using a DML INSERT statement. Go to the BigQuery console and paste in the folling DML INSERT:

   ```
    INSERT INTO `Cymbal_Pets.Fish_Tank_IOT_Data_Ingest` (event_timestamp,store_id,location,tank_id,temperature,ph_level,ammonia_ppm,nitrite_no2_ppm,nitrate_no3_ppm,salinity_ppm,event_metadata)
    VALUES (
     CURRENT_TIMESTAMP(),
     'store-789',
     STRUCT('Las Vegas', 'NV', '88901'),
     'salt_water_tank-101',
     56.2,
     7.8,
     0.25,
     0.1,
     30.0,
     35.0,
     JSON '{"sensor_type": "fish_sensor_3000", "sensor_id": "temp-sensor-abc", "reading_unit": "fahrenheit"}'
    );
   ```

2. Go back to your ServiceNow console, again make sure status is "Available", and wait ~30 seconds for the event to arrive. When the event arrives, note the valuable insights that Gemini provided to diagnose the fish tank issue and correct the problem.

   ![Screenshot 2025-04-24 at 2 39 01 PM](https://github.com/user-attachments/assets/fcfb30ab-d9ae-4f7c-b0ce-c325e6b48b94)


