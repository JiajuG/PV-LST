# PV-LST

This repository contains the data and processing frameworks for our study on the land surface temperature (LST) effects of photovoltaic power plants across the globe.

## Data Availability

The core datasets of this study are hosted on the **Google Earth Engine (GEE)** platform. Users can directly access and process these assets using the GEE cloud computing environment:

* **Global PV Station Dataset:** [projects/global-phenology/assets/GlobalPVStation](https://code.earthengine.google.com/?asset=projects/global-phenology/assets/GlobalPVStation)
* **Global PV Station Buffer Dataset:** [projects/global-phenology/assets/GlobalPVStationBuffer](https://code.earthengine.google.com/?asset=projects/global-phenology/assets/GlobalPVStationBuffer)
<img width="1043" height="836" alt="image" src="https://github.com/user-attachments/assets/1561b5d6-3485-4fa4-833d-233ce18fec53" />

### Usage Note
These assets can be loaded directly into the GEE Code Editor or through the Python API using the following code snippet:

```javascript
// Example for GEE JavaScript API
var pvStations = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStation");
var pvBuffers = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStationBuffer");
Map.addLayer(pvStations, {color: 'red'}, 'PV Stations');
