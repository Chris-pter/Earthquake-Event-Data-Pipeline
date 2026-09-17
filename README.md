# **Earthquake Event Data Engineering Pipeline · Microsoft Fabric**
An end-to-end data engineering pipeline built on Microsoft Fabric that collects real earthquake data from the USGS API, cleans and transforms it, and serves it to a Power BI dashboard, refreshed automatically every day and every 3 hours.

# **Overview**
The data is collected and processed through Microsoft Fabric, where it moves through a Bronze, Silver, Gold architecture. The processed data is then connected to a power BI semantic model and dashboard for analysis and reporting.

The project also includes a small geospatial component, using the earthquake coordinates to support location-based enrichment such as country identification.

**Stack**: Microsoft Fabric · PySpark · Delta Lake · USGS FDSNWS API · reverse geocoder

# **Architecture**
<img src= ./docs/>
## Medallion Layers
### Bronze-Raw Ingestion
### Notebook: Bronze

Calls the USGS Public API and saves the raw GeoJSON response as a date-partitioned JSON file in the Lakehouse, no transformations, raw data preserved exactly as received.

### Silver-Clean & Flatten
### Notebook: Silver

Reads the Bronze JSON and flattens the nested GeoJSON structure into a clean table/schema.

The transformation extracts information such as: 

* earthquake_id
* longitude
* latitude
* elevation
* title
* place_description
* sig
* mag
* magType
* time
* updated

The unix timestamps provided by the API are also converted into timestamp values, the cleaned data is stored in the Delta Table. The silver layer provides a cleaner and more consistent dataset for downstream processing.

### Gold-Enrich & Serve

### Notebook: Gold

Adds two enrichment columns on top of silver and upserts into **gold_events**. The gold layer prepares the data for analysis and reporting. The Silver data is filtered according to the pipeline date range and additional information is added.

Two main enrichments are performed.

### Significance Classification (sig_class)
* The USGS sig value is converted into a simple classification:

### Reverse Geocoding (country_code)
Each event's (latitude, longitude) in WGS84 (EPSG:4326) is resolved to a country code using the *reverse geocoder* library. It runs fully offline, no API key needed.

## Pipeline Orchestration
The pipeline is automated using **Microsoft Fabric Data Factory** and uses data variable to control the processing window. The pipeline uses a date-iterator loop, processing one day at a time across the date window, with Wait activities between each notebook to allow Spark compute to release its session before the next one starts.

### Dual Schedule

The daily run handles completeness, the intraday run keeps the dashboard current with the events that happened earlier today.

The pipeline uses Delta Lake **Merge** for incremental upsert processing. Existing earthquake records are updated in place, which is conceptually similar to SCD (Slowly Changing Dimension) Type 1 behavior, although the Gold table is an event table rather than a traditional dimension.
