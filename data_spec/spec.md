# NMFS Optics Data Cloud Migration: Minimum Viable Metadata Standard (MVS)

**Version:** 0.2 (Draft for DMC Review - GCP Optimized)
**Target Architecture:** Google Cloud Storage (GCS) + Serverless Query Engine (BigQuery / BigLake)
**Base Specification:** SpatioTemporal Asset Catalog (STAC) v1.0.0

## 1. Executive Summary

This document outlines the proposed metadata and storage standards for the NMFS-wide migration of optical data (images and video) to Google Cloud Platform (GCP). To avoid the costs and engineering bottlenecks of "lift-and-shift" database migrations, NMFS will adopt a **Decoupled Sidecar Architecture**.

Instead of a centralized, highly structured relational database, metadata will be written to lightweight JSON files stored immediately adjacent to the optics files in Google Cloud Storage. This enables serverless discovery via BigQuery while providing individual research groups maximum flexibility for their bespoke ancillary data.

## 2. Storage Prefix Convention

While cryptographic hashes prevent duplication, a purely flat bucket structure prevents manual inspection and complicates lifecycle management (e.g., migrating old surveys to Coldline or Archive storage classes). We propose a hybrid prefix approach.

**Proposed Standard Path:**
`gs://[bucket-name]/[Year]/[Region]/[Survey_ID]/[sha256_hash].[ext]`

*Example:*
`gs://nmfs-optics-archive/2024/NWFSC/WCG_2024/a7b9f3e4...8f.jpg`
`gs://nmfs-optics-archive/2024/NWFSC/WCG_2024/a7b9f3e4...8f.json` (Sidecar)

> **TODO (Data Management Committee):**
> Review and finalize the semantic levels of the prefix. Is `[Year]/[Region]/[Survey_ID]` sufficient for GCP lifecycle management policies and human-readability across all NMFS centers, or is a `[Program]` or `[Vessel]` level required? Keep this prefix as shallow as possible to optimize GCS listing operations.

## 3. The STAC Item Standard (Sidecar JSON)

Every optical asset uploaded to GCS MUST be accompanied by a STAC-compliant JSON sidecar file.

### 3.1 Core Requirements (Strictly Enforced)

For BigQuery serverless discovery to function NMFS-wide, the following STAC fields are strictly required and must adhere to standard formats:

1. **`id`**: Must match the exact `sha256` hash of the optical file.

2. **`geometry`**: Must be a valid GeoJSON point (Longitude, Latitude).

   * *Note: If a point is interpolated or generalized to a site bounding box due to missing telemetry, this must be noted in the properties.*

3. **`properties.datetime`**: Must be a valid ISO 8601 UTC timestamp (e.g., `2024-08-06T12:45:00Z`).

4. **`assets.visual`**: Must contain the exact Google Cloud Storage URI (`href`) of the target image/video.

### 3.2 Local Customization (Ancillary Data)

Individual groups maintain bespoke sensor suites and telemetry data. **Groups are not restricted in what they can upload.**

* **Point-in-time data:** Can be added directly to the `properties` block using local namespace prefixes (e.g., `nefsc:habcam_altitude`). BigQuery will automatically infer these fields during JSON schema detection.

* **High-frequency or complex data:** (e.g., continuous CTD casts, sonar backscatter, local Oracle DB dumps). Groups MUST NOT attempt to embed this in the STAC JSON. Instead, upload the raw file (CSV, NetCDF, Parquet) to a sub-prefix (e.g., `/ancillary/`) and link to it in the STAC `assets` dictionary.

## 4. The NMFS Core Vocabulary (`nmfs:` Extension)

To facilitate cross-center querying in BigQuery, a small subset of properties will be standardized under the `nmfs:` prefix. Where possible, these map to existing global ontologies like Darwin Core (DwC) or Climate and Forecast (CF) Conventions.

> **TODO (Data Management Committee):**
> Below is the proposed list of core `nmfs:` vocabulary. The DMC must finalize which of these are strictly *Required* vs *Optional*. The goal is to keep the required list as small as possible.

| Property Key | Type | Description / Example | External Ontology Mapping | 
| ----- | ----- | ----- | ----- | 
| `nmfs:region` | String | Sub-organization (e.g., "NWFSC", "PIFSC"). | *NMFS Internal* | 
| `nmfs:survey_id` | String | The official identifier for the survey cruise. | DwC: `eventID` | 
| `nmfs:platform` | String | Vessel, ROV, or AUV name (e.g., "Okeanos Explorer"). | BODC: `Platform type` | 
| `nmfs:instrument` | String | Type of camera/sensor (e.g., "DropCam", "HabCam"). | DwC: `samplingProtocol` | 
| `nmfs:depth_m` | Float | Depth of the instrument in meters at capture. | CF: `depth` | 
| `nmfs:altitude_m` | Float | Distance from instrument to the seafloor in meters. | CF: `height_above_sea_floor` | 
| `nmfs:spatial_qa` | String | Quality flag for geometry ("GPS", "USBL", "Interpolated", "Site-Level"). | DwC: `georeferenceRemarks` | 

## 5. Implementation Example

Below is a complete, valid STAC Item demonstrating both the enforced MVS structure and how a local group might integrate their specific ancillary data.

```json
{
  "stac_version": "1.0.0",
  "id": "a7b9f3e4b210984f18a2...8f",
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [-122.14562, 36.89211]
  },
  "bbox": [-122.14562, 36.89211, -122.14562, 36.89211],
  "properties": {
    "datetime": "2024-08-06T14:32:00Z",
    "nmfs:region": "NWFSC",
    "nmfs:survey_id": "WCG_2024",
    "nmfs:platform": "Shimada",
    "nmfs:depth_m": 150.5,
    "nmfs:spatial_qa": "USBL",
    
    "nwfsc_local:oracle_primary_key": "8847291",
    "nwfsc_local:pilot_name": "Smith, J"
  },
  "assets": {
    "visual": {
      "href": "gs://nmfs-optics-archive/2024/NWFSC/WCG_2024/a7b9f3e4b210984f18a2...8f.jpg",
      "type": "image/jpeg",
      "roles": ["data"]
    },
    "ctd_telemetry": {
      "href": "gs://nmfs-optics-archive/2024/NWFSC/WCG_2024/ancillary/site_42_CTD.csv",
      "type": "text/csv",
      "title": "Continuous CTD Cast Data",
      "roles": ["metadata"],
      "description": "Full cast telemetry associated with this dive site."
    }
  }
}
```

## 6. Next Steps for Adoption

1. **DMC Approval:** Finalize the GCS storage prefix and the required `nmfs:` vocabulary list.

2. **Upload Script Development:** Develop a reference Python script that local groups can adapt. This script will query a local database, hash the target file, generate the STAC JSON, and push both to the GCS bucket.

3. **BigQuery Integration:** Establish a BigLake connection or a BigQuery External Table pointing to the finalized GCS bucket structures. This will enable serverless SQL queries across all uploaded JSON metadata files natively within the GCP console.

