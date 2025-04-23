# BigQuery-Continuous-Queries-Fish-Tank-Health-IOT-Demo
This is an end-to-end demo using BigQuery continuous queries to address an unhealthy fish tank IOT sensor event with ServiceNow's Service Operations Workspace. This was demoed onstage at Google Cloud Next 2025 [recording](https://www.youtube.com/watch?v=SKv6bbwSL5I).

BigQuery continuous queries are SQL statements that run continuously, allowing you analyze incoming data in BigQuery in real time. You can insert the output rows produced by a continuous query into a BigQuery table or export them to Pub/Sub, Bigtable, or Spanner.

Documentation on BigQuery continuous queries can be found [HERE](https://cloud.google.com/bigquery/docs/continuous-queries-introduction), and a blog which provides context of BigQuery continuous queries can be found [HERE](https://cloud.google.com/blog/products/data-analytics/bigquery-continuous-queries-makes-data-analysis-real-time).


Let's assume our pet store has IOT sensors on each of our fish tanks to ensure our fish are happy and healthy with safe water conditions. 

It accomplishes this by streaming IOT sensor data to a table in BigQuery by way of a Pub/Sub topic and Pub/Sub to BigQuery subscription.

From here, a continuous query is running which processes the incoming IOT sensor data in real time, checks if the water conditions are unhealthy, and if unhealthy calls out to a Gemini 2.0 flash model to determine the possible cause of the issue and a quick remedy. 

The continuous query then writes this message to another Pub/Sub topic, which is received by ServiceNow’s Workflow Data Fabric that creates a service ticket for the pet store team to immediately rectify the problem with the fish tank.
