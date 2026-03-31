# PV-LST

This repository contains the data and processing frameworks for our study on the land surface temperature (LST) effects of photovoltaic power plants across the globe.

## Data Availability

The core datasets of this study are hosted on the **Google Earth Engine (GEE)** platform. Users can directly access and process these assets using the GEE cloud computing environment:

* **Global PV Station Dataset:** [projects/global-phenology/assets/GlobalPVStation](https://code.earthengine.google.com/?asset=projects/global-phenology/assets/GlobalPVStation)
* **Global PV Station Buffer Dataset:** [projects/global-phenology/assets/GlobalPVStationBuffer](https://code.earthengine.google.com/?asset=projects/global-phenology/assets/GlobalPVStationBuffer)
* <img width="1043" height="836" alt="image" src="https://github.com/user-attachments/assets/1561b5d6-3485-4fa4-833d-233ce18fec53" />


## 2. GEE Code  (JavaScript)

The following script compares daytime and nighttime LST for a specific PV facility using the **MYD11A2** product. You can copy and paste it directly into the [GEE Code Editor](https://code.earthengine.google.com/).

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
```
<img width="1668" height="696" alt="image" src="https://github.com/user-attachments/assets/d0529319-3f86-4a45-9144-ce4538ba3f57" />

Batch Processing of LST Data (Multiple Stations):
The following script performs batch extraction of LST statistics for all PV stations within the dataset.
```javascript
/**
 * Batch processing of MODIS LST (MYD11A2) for multiple PV stations.
 * This script extracts daytime and nighttime median LST (Celsius) 
 * for the first 500 sites and exports the results to Google Drive.
 */

// ===== 1. Load PV Station and Buffer Assets =====
var polygons = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStation");
var polygons0 = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStationBuffer");

// Get a list of unique site IDs from the collection
var ids = polygons.aggregate_array('site_id');

// ===== 2. Define LST Loading Function =====
/**
 * Loads MYD11A2 LST collection filtered by geometry and date range.
 * @param {ee.Geometry} geometry The spatial area of interest.
 * @return {ee.ImageCollection} Filtered MODIS LST collection.
 */
function loadLST(geometry) {
  return ee.ImageCollection('MODIS/061/MYD11A2')
    .filterBounds(geometry)
    .filterDate('2005-01-01', '2024-12-31')
    .select(['LST_Day_1km', 'LST_Night_1km']);
}

// ===== 3. Define Single-Station Processing Function =====
/**
 * Processes a single station to extract LST time-series data.
 * @param {String} uid The unique site_id.
 * @return {ee.FeatureCollection} Collection of LST statistics per date.
 */
var processOneStation = function(uid) {
  var target = polygons.filter(ee.Filter.eq('site_id', uid));
  var collection = loadLST(target);

  var stats = collection.map(function(img) {
    var date = img.date().format('YYYY-MM-dd');

    // Unit Conversion: Scale factor (0.02) and Kelvin to Celsius (-273.15)
    var lst_day_c = img.select('LST_Day_1km').multiply(0.02).subtract(273.15);
    var lst_night_c = img.select('LST_Night_1km').multiply(0.02).subtract(273.15);

    // Spatial aggregation: Calculate median LST within the station boundary
    var reduced = ee.Image.cat([
      lst_day_c.rename('LST_Day_C'),
      lst_night_c.rename('LST_Night_C')
    ]).reduceRegion({
      reducer: ee.Reducer.median(),
      geometry: target.geometry(),
      scale: 1000, // MODIS 1km resolution
      maxPixels: 1e9
    });

    return ee.Feature(null, {
      'site_id': uid,
      'date': date,
      'LST_Day_C': reduced.get('LST_Day_C'),
      'LST_Night_C': reduced.get('LST_Night_C')
    });
  }).filter(ee.Filter.notNull(['LST_Day_C', 'LST_Night_C']))
    .sort('date');

  return stats;
};

// ===== 4. Client-side Loop for Batch Processing (Indices 0-500) =====
// Note: We use .getInfo() to iterate on the client side for independent task queuing
var resultsList = [];
var siteIds = ids.getInfo().slice(0, 500);

print('Total sites to process:', siteIds.length);

siteIds.forEach(function(uid) {
  var result = processOneStation(uid);
  resultsList.push(result);
});

// ===== 5. Merge Results and Export to CSV =====
// Use ee.FeatureCollection() to combine the list of collections
var allResults = ee.FeatureCollection(resultsList).flatten();

Export.table.toDrive({
  collection: allResults,
  description: 'Aqua_LST_DayNight_Median_2005_2024_Part1',
  folder: 'Aqua_LST_Analysis',
  fileFormat: 'CSV',
  selectors: ['site_id', 'date', 'LST_Day_C', 'LST_Night_C']
});

print('🚀 Batch processing started. Check the Tasks tab to confirm the export.');
```
