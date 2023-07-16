---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Education
======
* B.E. in Computer Science (CGPA: 8.22/10), Thapar University, 2019

Work experience
======
* Jan 2022 - Present: Data Engineer @ Doodle
  * Researched and Implemented ZSTD compression method in data lake over snappy saving 35% in storage cost, 30% in data scan cost for Athena. All resulting in 20% faster queries
  * Created a Data Model for millions of daily records ingestion from kafka events for frontend activity on doodle’s website. This powers user churn and conversion analysis and feature activations
  * Created a scalable solution for ingestion of user activity from kafka topics to database using kafka connect and airflow which empowered in simplifying the dataflow and helped tracking user’s lifetime value
  * Researched & Created a solution to track and reduce costs of Athena and S3 by solving small file problem for more effective processing of data as well improve the compression of data in parquet files
  * Implemented Apache Iceberg to support updation and deletion for GDPR compliance in data lake (S3) using Spark (EMR)
  * Developed custom airflow operator for specific repeated use-cases like mirroring data from athena to redshift among others
  * Implemented dbt core to enable data democratisation and create accessible data docs for the entire organisation to leverage and support on data-driven culture

* August 2021 - Dec 2021: Data Engineer @ Dresslife
  * Created data pipelines from kafka for MAB for choosing a model from multiple ML models based on its performance in an online fashion based on conversion rate and average order value which helped improve revenue uplift from 3% to 4.3%
  * Created multiple variants of SR-GNN (Session based GNN) and successfully deployed them to EKS on AWS and helped delivered 3% revenue uplift for fashion retail in sales and non sale situations

* July 2019 - July 2021: Data Engineer @ MAQ Software
  * Developed almost real time Power BI dashboards for Exec team to look at KPIs for various domains using apache spark and azure data lake
  * Create Models that served multiple use cases and KPI predictions for business to better understand the data and help them make informative decisions
  * Setup alerts and data quality alerts for various data flows to monitor SLAs and incident management from the ground up using azure application insights
  * Work on MLFlow and concept drift (based on wasserstein distance) for ML-Ops for retraining alerts and triggers. Use Memory profiler and statistical profiler to log impact on real time services on the retraining model. Extended MLOps to include Sequential A/B testing to test model performance before exposing it to all users.
  
* Jan 2019 - Jun 2019: Intern @ MAQ Software
  * Developed Power BI dashboards and data pipelines
  
Skills
======
* C, Python, SQL and Java
* AWS and Azure
* Power BI, Metabase and Popsql
* A/B Testing, ML-Ops, Predictive Analytics and NLP
* Spark, Flink, Airflow and Kafka
* Grafana, Sentry and Elastic
* Kubernetes and Docker 

Publications
======
  <ul>{% for post in site.publications %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>