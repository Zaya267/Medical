# Medical Data Engineering – Diagnosis & Treatment Analytics Pipeline

## Overview
ETL + analytics pipeline for analyzing patient symptoms, diagnoses, treatments, including geospatial clustering.

## Environment Setup (Free Tier)
- AWS S3 buckets: medical-raw, medical-staging, medical-curated
- Snowflake warehouse: XSMALL, AUTO_SUSPEND = 60s
- Database: MEDICAL_DB, schema: PUBLIC

## Step 1: File Format & Stage
```sql
CREATE FILE FORMAT MEDICAL_CSV_FORMAT
  TYPE = 'CSV'
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  SKIP_HEADER = 1;

CREATE STAGE MEDICAL_STAGE
  URL='s3://medical-raw/'
  FILE_FORMAT = MEDICAL_CSV_FORMAT
  CREDENTIALS=(AWS_KEY_ID='<ACCESS_KEY>' AWS_SECRET_KEY='<SECRET_KEY>');
```

## Step 2: RAW Table
```sql
CREATE OR REPLACE TABLE RAW_PATIENTS (
  PATIENT_ID STRING,
  SYMPTOMS STRING,
  DIAGNOSIS STRING,
  TREATMENT STRING,
  PATIENT_LOCATION STRING
);
```

## Step 3: Load CSV
```sql
COPY INTO RAW_PATIENTS
FROM @MEDICAL_STAGE
FILE_FORMAT = (FORMAT_NAME = MEDICAL_CSV_FORMAT)
ON_ERROR = 'CONTINUE';
```

## Step 4: Staging Table
```sql
CREATE OR REPLACE TABLE STG_PATIENTS AS
SELECT
  PATIENT_ID,
  SYMPTOMS,
  DIAGNOSIS,
  TREATMENT,
  TRY_TO_GEOGRAPHY(PATIENT_LOCATION) AS LOCATION_GEOG
FROM RAW_PATIENTS;
```

## Step 5: Fact & Dimension Tables
```sql
CREATE OR REPLACE TABLE DIM_PATIENT (
  PATIENT_ID STRING PRIMARY KEY
);

CREATE OR REPLACE TABLE DIM_SYMPTOMS (
  SYMPTOMS STRING PRIMARY KEY
);

CREATE OR REPLACE TABLE FACT_DIAGNOSES (
  PATIENT_ID STRING REFERENCES DIM_PATIENT(PATIENT_ID),
  SYMPTOMS STRING REFERENCES DIM_SYMPTOMS(SYMPTOMS),
  DIAGNOSIS STRING,
  TREATMENT STRING,
  LOCATION_GEOG GEOGRAPHY
);
```

## Step 6: Business Rules
```sql
-- Remove duplicate or incomplete rows
DELETE FROM STG_PATIENTS WHERE SYMPTOMS IS NULL OR DIAGNOSIS IS NULL;
```

## Step 7: Incremental Load
```sql
CREATE OR REPLACE STREAM STG_PATIENTS_STREAM ON TABLE STG_PATIENTS;

CREATE OR REPLACE TASK LOAD_FACT_DIAGNOSES
  WAREHOUSE = XSMALL
  SCHEDULE = 'USING CRON 0 * * * * UTC'
AS
INSERT INTO FACT_DIAGNOSES (PATIENT_ID, SYMPTOMS, DIAGNOSIS, TREATMENT, LOCATION_GEOG)
SELECT PATIENT_ID, SYMPTOMS, DIAGNOSIS, TREATMENT, LOCATION_GEOG
FROM STG_PATIENTS_STREAM;
```

## Step 8: Geospatial Analytics
```sql
-- Example: Patients within 5km of clinic
SELECT *
FROM STG_PATIENTS
WHERE ST_DISTANCE(LOCATION_GEOG, ST_POINT(<CLINIC_LONG>, <CLINIC_LAT>)) < 5000;
```

## Step 9: Dashboard Creation
- **Snowflake Worksheets:**
  - Create views for key metrics:
    ```sql
    CREATE OR REPLACE VIEW V_PATIENT_TREATMENTS AS
    SELECT DIAGNOSIS, TREATMENT, COUNT(*) AS NUM_PATIENTS
    FROM FACT_DIAGNOSES
    GROUP BY DIAGNOSIS, TREATMENT;
    ```
  - Create geospatial heatmap view:
    ```sql
    CREATE OR REPLACE VIEW V_PATIENT_HOTSPOTS AS
    SELECT LOCATION_GEOG, COUNT(*) AS PATIENT_COUNT
    FROM FACT_DIAGNOSES
    GROUP BY LOCATION_GEOG;
    ```
- **Snowflake Snowsight Dashboards:**
  1. Use "Charts" to plot treatment counts by diagnosis.
  2. Use "Map Charts" for V_PATIENT_HOTSPOTS to visualize patient density.
  3. Combine multiple charts into a dashboard for monitoring patient clusters and treatment trends.
- **Optional Export:** Connect Snowflake to Tableau or Power BI for advanced visualization.

