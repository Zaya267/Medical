# Medical Data Engineering – Diagnosis & Treatment Analytics Pipeline

## Overview
ETL + analytics pipeline for analyzing patient symptoms, diagnoses, treatments, including geospatial clustering.

## Architecture
RAW → Staging → Fact/Dimension → Analytics → Geospatial Insights

## Data Sources
- Mockaroo: synthetic patient datasets with location
- Kaggle Public Health datasets: https://www.kaggle.com/datasets

## Environment Setup (Free Tier)
1. AWS S3 buckets: medical-raw, medical-staging, medical-curated
2. Snowflake: XSMALL warehouse, AUTO_SUSPEND = 60s, Database MEDICAL_DB, schema PUBLIC

## Steps
1. Create file format and stage in Snowflake pointing to S3
2. Create RAW table RAW_PATIENTS and load CSV
3. Create staging table STG_PATIENTS with proper types
4. Create dimension and fact tables: DIM_PATIENT, DIM_SYMPTOMS, FACT_DIAGNOSES
5. Apply business rules: normalize symptoms, remove duplicates, handle missing data
6. Create STREAM and TASK for incremental loads
7. Geospatial: PATIENT_LOCATION → GEOGRAPHY, cluster analysis
8. Dashboard: treatment trends by cluster

## Cost Control
- XSMALL warehouse
- AUTO_SUSPEND = 60s
- TASK scheduled hourly

## Scripts
01_create_file_format_and_stage.sql
02_create_raw_table.sql
03_load_raw.sql
04_create_staging_table.sql
05_create_fact_dim_tables.sql
06_apply_business_rules.sql
07_create_stream_and_task.sql
08_geospatial_analysis.sql