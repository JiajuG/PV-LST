# PV-LST

This repository contains the data and processing frameworks for our study on the land surface temperature (LST) effects of photovoltaic power plants across the globe.

## Data Availability

The core datasets of this study are hosted on the **Google Earth Engine (GEE)** platform. Users can directly access and process these assets using the GEE cloud computing environment:

* **Global PV Station Dataset:** [projects/global-phenology/assets/GlobalPVStation](https://code.earthengine.google.com/?asset=projects/global-phenology/assets/GlobalPVStation)
* **Global PV Station Buffer Dataset:** [projects/global-phenology/assets/GlobalPVStationBuffer](https://code.earthengine.google.com/?asset=projects/global-phenology/assets/GlobalPVStationBuffer)
* <img width="1043" height="836" alt="image" src="https://github.com/user-attachments/assets/1561b5d6-3485-4fa4-833d-233ce18fec53" />


## LST Processing Example (GEE JavaScript API)

The following script demonstrates how to extract and compare the median daytime and nighttime LST for individual PV facilities using the 8-day composite Aqua MODIS product (**MYD11A2**). 

You can copy and paste this script directly into the [GEE Code Editor](https://code.earthengine.google.com/).

```javascript
/**
 * Comparison of Daytime and Nighttime Land Surface Temperature (LST) 
 * for a specific PV station using Aqua MODIS (MYD11A2).
 */

// ===== 1. Load PV Station Boundary Data =====
var polygons = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStation");

// Select a representative site for time-series analysis (e.g., the 901st site)
var firstID = polygons.aggregate_array('site_id').get(900);
var target = polygons.filter(ee.Filter.eq('site_id', firstID));

// ===== 2. Load Aqua MODIS LST (MYD11A2) Collection =====
var lstCol = ee.ImageCollection('MODIS/061/MYD11A2')
  .filterBounds(target)
  .filterDate('2010-01-01', '2020-12-31')
  .select(['LST_Day_1km', 'LST_Night_1km']);

// ===== 3. Extract Median Values and Convert to Celsius =====
var lstStats = lstCol.map(function(img) {
  var date = img.date().format('YYYY-MM-dd');
  
  // Apply scale factor (0.02) and convert Kelvin to Celsius
  var dayImg = img.select('LST_Day_1km').multiply(0.02).subtract(273.15);
  var nightImg = img.select('LST_Night_1km').multiply(0.02).subtract(273.15);

  var stat = ee.Image.cat([dayImg.rename('LST_Day_C'), nightImg.rename('LST_Night_C')])
    .reduceRegion({
      reducer: ee.Reducer.median(),
      geometry: target.geometry(),
      scale: 1000,
      maxPixels: 1e9
    });

  return ee.Feature(null, {
    'date': date,
    'LST_Day_C': stat.get('LST_Day_C'),
    'LST_Night_C': stat.get('LST_Night_C')
  });
})
.filter(ee.Filter.notNull(['LST_Day_C', 'LST_Night_C']))
.sort('date');

// ===== 4. Generate Multi-temporal Chart (Day vs. Night) =====
var lstCompareChart = ui.Chart.feature.byFeature(lstStats, 'date')
  .setChartType('LineChart')
  .setOptions({
    title: 'Aqua LST Day vs Night Median Time Series (°C, MYD11A2)',
    hAxis: {title: 'Date', titleTextStyle: {italic: false}},
    vAxis: {title: 'Land Surface Temperature (°C)', titleTextStyle: {italic: false}},
    lineWidth: 2,
    pointSize: 1,
    series: {
      0: {color: '#d62728', label: 'Daytime Median LST'},
      1: {color: '#1f77b4', label: 'Nighttime Median LST'}
    }
  });

// ===== 5. Output and Visualization =====
print('📈 Time Series Analysis:', lstCompareChart);
Map.centerObject(target, 12);
Map.addLayer(target, {color: 'FF0000'}, 'Target PV Station');

### Usage Note
These assets can be loaded directly into the GEE Code Editor or through the Python API using the following code snippet:

```javascript
// Example for GEE JavaScript API
var pvStations = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStation");
var pvBuffers = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStationBuffer");
Map.addLayer(pvStations, {color: 'red'}, 'PV Stations');
