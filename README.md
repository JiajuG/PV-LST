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
Batch Processing of NDVI Data (One Stations):
```javascript
/**
 * NDVI Time-Series Analysis for PV Stations.
 * This script integrates Terra (MOD13Q1) and Aqua (MYD13Q1) datasets to analyze 
 * vegetation dynamics (Mean, Median, StdDev) for a selected PV site.
 */

// ===== 1. Load PV Station Boundary Data =====
var polygons = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStation");

// Select a representative site for analysis (e.g., index 900)
var firstID = polygons.aggregate_array('site_id').get(900);
print('Selected Site ID:', firstID);
var target = polygons.filter(ee.Filter.eq('site_id', firstID));

// ===== 2. Helper Function: Load and Pre-process NDVI with Source Tags =====
/**
 * Processes MODIS NDVI collections: scales values, masks poor quality pixels, and adds source tags.
 * @param {String} collectionId MODIS Collection ID (MOD13Q1 or MYD13Q1).
 * @param {String} sourceTag Label to identify the satellite source (Terra or Aqua).
 * @return {ee.ImageCollection} Cleaned and tagged NDVI collection.
 */
function loadFilteredNDVI(collectionId, sourceTag) {
  return ee.ImageCollection(collectionId)
    .filterBounds(target)
    .filterDate('2010-01-01', '2024-12-31')
    .map(function(img) {
      // Scale factor (0.0001) to convert raw values to standard NDVI range [-1, 1]
      var ndvi = img.select('NDVI').multiply(0.0001).rename('NDVI');
      
      // Quality Assurance (QA) masking: Keep only "Good Quality" pixels (SummaryQA == 0)
      var qa = img.select('SummaryQA');
      var mask = qa.eq(0);
      
      ndvi = ndvi.updateMask(mask).copyProperties(img, ['system:time_start']);
      return ndvi.set('source', sourceTag);
    });
}

// ===== 3. Load Terra and Aqua Collections and Merge =====
var mod13 = loadFilteredNDVI('MODIS/061/MOD13Q1', 'Terra');
var myd13 = loadFilteredNDVI('MODIS/061/MYD13Q1', 'Aqua');

// Combine both datasets and sort by time to increase temporal frequency
var combined = mod13.merge(myd13).sort('system:time_start');

// ===== 4. Extract Raw NDVI Median Points with Source Attribution =====
var ndviPoints = combined.map(function(img) {
  var date = img.date().format('YYYY-MM-dd');
  var median = img.reduceRegion({
    reducer: ee.Reducer.median(),
    geometry: target.geometry(),
    scale: 250, // MODIS NDVI native resolution
    maxPixels: 1e9
  }).get('NDVI');

  return ee.Feature(null, {
    'date': date,
    'NDVI': median,
    'source': img.get('source')
  });
}).filter(ee.Filter.notNull(['NDVI'])).sort('date');

// ===== 5. Spatial Statistics: Mean, Median, and Standard Deviation =====
var ndviStats = combined.map(function(img) {
  var date = img.date().format('YYYY-MM-dd');
  var stats = img.reduceRegion({
    // Combining multiple reducers for efficient spatial computation
    reducer: ee.Reducer.mean()
              .combine({reducer2: ee.Reducer.median(), sharedInputs: true})
              .combine({reducer2: ee.Reducer.stdDev(), sharedInputs: true}),
    geometry: target.geometry(),
    scale: 250,
    maxPixels: 1e9
  });

  return ee.Feature(null, {
    'date': date,
    'NDVI_mean': stats.get('NDVI_mean'),
    'NDVI_median': stats.get('NDVI_median'),
    'NDVI_stdDev': stats.get('NDVI_stdDev')
  });
}).filter(ee.Filter.notNull(['NDVI_mean', 'NDVI_median', 'NDVI_stdDev']))
  .sort('date');

// ===== 6. Scatter Chart: Terra vs. Aqua Comparison =====
var scatter = ui.Chart.feature.groups({
  features: ndviPoints,
  xProperty: 'date',
  yProperty: 'NDVI',
  seriesProperty: 'source'
})
.setChartType('ScatterChart')
.setOptions({
  title: 'NDVI Time-Series: Terra vs. Aqua Data Points',
  hAxis: {title: 'Date'},
  vAxis: {title: 'NDVI Value'},
  pointSize: 3,
  lineWidth: 0,
  series: {
    0: {color: '#d62728'}, // Terra (Red)
    1: {color: '#1f77b4'} // Aqua (Blue)
  }
});

// ===== 7. Line Chart: Spatial Statistics Trends =====
var line = ui.Chart.feature.byFeature(ndviStats, 'date')
  .setChartType('LineChart')
  .setOptions({
    title: 'NDVI Spatial Statistics (Mean, Median, StdDev)',
    hAxis: {title: 'Date'},
    vAxis: {title: 'NDVI'},
    lineWidth: 2,
    pointSize: 0,
    series: {
      0: {color: '#333333', label: 'Spatial Mean'},
      1: {color: '#1f77b4', label: 'Spatial Median'},
      2: {color: '#9467bd', label: 'Spatial StdDev'}
    }
  });

// ===== 8. Output Visualization =====
print('📈 NDVI Comparison (Terra vs. Aqua):', scatter);
print('📈 NDVI Statistical Curves:', line);
```
<img width="1498" height="689" alt="image" src="https://github.com/user-attachments/assets/8abcf0c4-c91a-4e08-a473-9b5236d674d2" />

Batch Processing of NDVI Data (Multiple Stations):
```javascript

/**
 * Batch processing of MODIS NDVI (MOD13Q1 & MYD13Q1) for PV station buffers.
 * This script integrates Terra and Aqua datasets to extract vegetation statistics 
 * (Mean, Median, StdDev) for the first 500 sites between 2005 and 2010.
 */

// ===== 1. Load PV Station Buffer Assets =====
var polygons = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStationBuffer");
var ids = polygons.aggregate_array('site_id');

// ===== 2. Define Helper Function: Load NDVI with QA Masking and Source Tagging =====
/**
 * Processes MODIS NDVI collections: scales values, masks low-quality pixels, and adds source tags.
 * @param {String} collectionId MODIS Collection ID (MOD13Q1 or MYD13Q1).
 * @param {String} sourceTag Label to identify the satellite source (MOD or MYD).
 * @param {ee.FeatureCollection} geometry The spatial filter area.
 * @return {ee.ImageCollection} Cleaned and tagged NDVI collection.
 */
function loadFilteredNDVI(collectionId, sourceTag, geometry) {
  return ee.ImageCollection(collectionId)
    .filterBounds(geometry)
    .filterDate('2005-01-01', '2010-12-31')
    .map(function(img) {
      // Scale factor (0.0001) to convert raw values to standard NDVI range [-1, 1]
      var ndvi = img.select('NDVI').multiply(0.0001).rename('NDVI');
      
      // Quality Assurance (QA) masking: Keep only high-quality pixels (SummaryQA == 0)
      var qa = img.select('SummaryQA');
      var mask = qa.eq(0); 
      
      return ndvi
        .updateMask(mask)
        .copyProperties(img, ['system:time_start'])
        .set('source', sourceTag);
    });
}

// ===== 3. Define Single-Station Processing Function (Terra + Aqua) =====
/**
 * Processes a single station to extract NDVI spatial statistics time-series.
 * @param {String} uid The unique site_id.
 * @return {ee.FeatureCollection} Collection of NDVI statistics per date.
 */
var processOneStation = function(uid) {
  var target = polygons.filter(ee.Filter.eq('site_id', uid));

  // Load both Terra (MOD) and Aqua (MYD) data to increase temporal frequency
  var mod = loadFilteredNDVI('MODIS/061/MOD13Q1', 'MOD', target);
  var myd = loadFilteredNDVI('MODIS/061/MYD13Q1', 'MYD', target);

  var combined = mod.merge(myd).sort('system:time_start');

  // Calculate spatial statistics (Mean, Median, StdDev) for each image
  var stats = combined.map(function(img) {
    var date = img.date().format('YYYY-MM-dd');
    var source = img.get('source');

    var regionStats = img.reduceRegion({
      reducer: ee.Reducer.mean()
                .combine({reducer2: ee.Reducer.median(), sharedInputs: true})
                .combine({reducer2: ee.Reducer.stdDev(), sharedInputs: true}),
      geometry: target.geometry(), // GEE handles automatic projection transformation
      scale: 250, // MODIS NDVI native resolution
      maxPixels: 1e9
    });

    return ee.Feature(null, {
      'site_id': uid,
      'date': date,
      'source': source,
      'NDVI_mean': regionStats.get('NDVI_mean'),
      'NDVI_median': regionStats.get('NDVI_median'),
      'NDVI_stdDev': regionStats.get('NDVI_stdDev')
    });
  }).filter(ee.Filter.notNull(['NDVI_median'])).sort('date');

  return stats;
};

// ===== 4. Client-side Loop Control (Indices 0-500) =====
var resultsList = [];
var siteIds = ids.getInfo().slice(0, 500);

print('Total sites to be processed:', siteIds.length);

siteIds.forEach(function(uid) {
  print('⏳ Processing station:', uid);
  var result = processOneStation(uid);
  resultsList.push(result);
});

// ===== 5. Merge All Results and Export to CSV =====
// Flattens the list of FeatureCollections into a single FeatureCollection for export
var allResults = ee.FeatureCollection(resultsList).flatten();

Export.table.toDrive({
  collection: allResults,
  description: 'MODIS_NDVI_Merged_2005_2010_Part1',
  folder: 'NDVI_Analysis',
  fileFormat: 'CSV'
});

print('🚀 Batch processing for NDVI initiated. Please monitor the Tasks tab.');
```

Batch Processing of Albedo Data (Multiple Stations):
<img width="1710" height="693" alt="image" src="https://github.com/user-attachments/assets/ccdf6bc4-101a-402c-b189-1e7683135a76" />
```javascript
/**
 * Time-Series Analysis of Surface Albedo for PV Stations.
 * This script extracts the median White-sky Shortwave Albedo (WSA) 
 * from the MODIS MCD43A3 product for a selected PV site.
 */

// ===== 1. Load PV Station Boundary Data =====
var polygons = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStation");

// Select a representative site for analysis (e.g., index 900)
var firstID = polygons.aggregate_array('site_id').get(900);
var target = polygons.filter(ee.Filter.eq('site_id', firstID));

// ===== 2. Load MODIS Albedo (MCD43A3) Collection =====
// Selecting the White-sky Albedo (WSA) for the shortwave band (0.3–5 μm)
var albedoCol = ee.ImageCollection('MODIS/061/MCD43A3')
  .filterBounds(target)
  .filterDate('2010-01-01', '2020-12-31')
  .select('Albedo_WSA_shortwave');

// ===== 3. Extract Median Albedo Time-Series =====
var albedoStats = albedoCol.map(function(img) {
  var date = img.date().format('YYYY-MM-dd');
  
  // Spatial aggregation: Calculate the median albedo within the station boundary
  var stat = img.reduceRegion({
    reducer: ee.Reducer.median(),
    geometry: target.geometry(),
    scale: 500, // MCD43A3 native resolution is 500m
    maxPixels: 1e9
  });

  return ee.Feature(null, {
    'date': date,
    'Albedo_median': stat.get('Albedo_WSA_shortwave')
  });
}).filter(ee.Filter.notNull(['Albedo_median'])) // Filter out cloud-contaminated or missing data
  .sort('date');

// ===== 4. Generate Time-Series Line Chart =====
var albedoLineChart = ui.Chart.feature.byFeature(albedoStats, 'date')
  .setChartType('LineChart')
  .setOptions({
    title: 'White-sky Shortwave Albedo Median (MCD43A3)',
    hAxis: {
      title: 'Date',
      titleTextStyle: {italic: false}
    },
    vAxis: {
      title: 'Albedo (0.3–5 μm)', 
      minValue: 0, 
      maxValue: 1,
      titleTextStyle: {italic: false}
    },
    lineWidth: 2,
    pointSize: 1, // Added small points for better visibility of data frequency
    series: {
      0: {color: '#2ca02c', label: 'Median Albedo'} // Professional green hex code
    },
    curveType: 'function' // Optional: Smooths the trend line
  });

// ===== 5. Output Visualization =====
print('📈 Median Albedo Time Series:', albedoLineChart);
Map.centerObject(target, 12);
Map.addLayer(target, {color: 'FF0000'}, 'Target PV Station');
```
Batch Processing of LE Data (Multiple Stations):
```javascript
/**
 * Batch processing of MODIS Latent Heat Flux (LE) for PV stations.
 * This script extracts the median LE (scaled to W/m^2) using MOD16A2GF 
 * for the first 500 sites and exports a time-series CSV to Google Drive.
 */

// ===== 1. Load PV Station Buffer Assets =====
var polygons = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStationBuffer");
var ids = polygons.aggregate_array('site_id');

// ===== 2. Define LE Loading and Pre-processing Function =====
/**
 * Loads MOD16A2GF LE band, applies scale factor, and masks invalid pixels.
 * @param {ee.Geometry} geometry Spatial filter area.
 * @return {ee.ImageCollection} Cleaned Latent Heat Flux collection.
 */
function loadLE(geometry) {
  return ee.ImageCollection('MODIS/061/MOD16A2GF')
    .filterBounds(geometry)
    .filterDate('2005-01-01', '2024-12-31')
    .select('LE')
    .map(function(img) {
      // Apply scale factor (0.1) and mask fill value (32767) or non-positive values
      return img
        .multiply(0.1)
        .updateMask(img.neq(32767).and(img.gt(0)))
        .set('system:time_start', img.get('system:time_start'));
    });
}

// ===== 3. Single-Station Processing Function (LE Median Time Series) =====
/**
 * Processes a single station to extract median LE time-series data.
 * @param {String} uid The unique site_id.
 * @return {ee.FeatureCollection} Collection of LE statistics per date.
 */
var processOneStation = function(uid) {
  var target = polygons.filter(ee.Filter.eq('site_id', uid));
  var geometry = target.geometry(); 
  var collection = loadLE(geometry);

  var stats = collection.map(function(img) {
    var date = ee.Date(img.get('system:time_start')).format('YYYY-MM-dd');
    
    // Spatial aggregation: Calculate median LE within the buffer
    var stat = img.reduceRegion({
      reducer: ee.Reducer.median(),
      geometry: geometry,
      scale: 500, // MOD16A2GF native resolution is 500m
      maxPixels: 1e9
    });
    
    return ee.Feature(null, {
      'site_id': uid,
      'date': date,
      'LE_median': stat.get('LE')
    });
  }).filter(ee.Filter.notNull(['LE_median']))
    .sort('date');

  return stats;
};

// ===== 4. Client-side Loop Control (First 500 Stations) =====
var resultsList = [];
var siteIds = ids.getInfo().slice(0, 500);

siteIds.forEach(function(uid) {
  try {
    print('⏳ Processing station:', uid);
    var result = processOneStation(uid);
    resultsList.push(result);
  } catch (e) {
    print('⚠️ Skipping failed station:', uid);
  }
});

// ===== 5. Merge Results and Export to CSV =====
// Each row includes: site_id, date, and LE_median
var allResults = ee.FeatureCollection(resultsList).flatten();
print('✅ Final FeatureCollection Preview (First 5):', allResults.limit(5));

Export.table.toDrive({
  collection: allResults,
  description: 'MODIS_LE_Median_2005_2024_Part1',
  folder: 'MODIS_LE_Analysis',
  fileFormat: 'CSV'
});

print('🚀 Batch processing for LE initiated. Please monitor the Tasks tab.');
```
Batch Processing of ERA-5 Data (Multiple Stations):
```javascript
/**
 * Batch processing of ERA5-Land Monthly Aggregated Data for selected PV sites.
 * This script extracts 12 hydro-meteorological variables (2005-2024) 
 * for a predefined list of station UIDs and exports the results to Google Drive.
 */

// ===== 1. Load PV Station Vector Assets =====
var polygons = ee.FeatureCollection("projects/global-phenology/assets/GlobalPVStation");

// ===== 2. Define ERA5-Land Data Loading Function =====
function loadERA5Land(geometry) {
  return ee.ImageCollection("ECMWF/ERA5_LAND/MONTHLY_AGGR")
    .filterBounds(geometry)
    .filterDate('2005-01-01', '2024-12-31');
}

// ===== 3. Define Target Variables (Bands) =====
var bands = [
  'dewpoint_temperature_2m',
  'temperature_2m',
  'skin_reservoir_content',
  'surface_latent_heat_flux_sum',
  'surface_net_solar_radiation_sum',
  'surface_sensible_heat_flux_sum',
  'surface_solar_radiation_downwards_sum',
  'potential_evaporation_sum',
  'total_precipitation_sum',
  'total_evaporation_sum',
  'temperature_2m_max',
  'snow_cover'
];

// ===== 4. Specify the List of Target Site UIDs =====
var selected_uids = [
  'PV10385', 'PV1076', 'PV10784', 'PV10802', 'PV10944', // ... (rest of your list)
  'PV903', 'PV9245'
];

// ===== 5. Single-Station Processing Function =====
/**
 * Extracts monthly ERA5-Land variables for a specific site.
 * @param {String} uid The unique site_id.
 * @return {ee.FeatureCollection} Monthly meteorological statistics.
 */
var processOneStation = function(uid) {
  var target = polygons.filter(ee.Filter.eq('site_id', uid));
  var geometry = target.geometry().bounds(); // Use bounds to accelerate spatial queries

  var collection = loadERA5Land(geometry);

  var stats = collection.map(function(img) {
    var date = img.date().format('YYYY-MM-dd');

    // Extract mean values for all specified bands within the site geometry
    var reduced = img.select(bands).reduceRegion({
      reducer: ee.Reducer.mean(),
      geometry: geometry,
      scale: 11132, // ERA5-Land native resolution (0.1 arc-degree ≈ 11km)
      maxPixels: 1e9
    });

    // Return a feature with the UID and date as primary keys
    return ee.Feature(null, reduced)
             .set('uid', uid)
             .set('date', date);
  }).filter(ee.Filter.notNull(['temperature_2m'])) // Filter out empty records
    .sort('date');

  return stats;
};

// ===== 6. Execute Batch Processing via Client-side Loop =====
var resultsList = [];
selected_uids.forEach(function(uid) {
  print('⏳ Processing station:', uid);
  var result = processOneStation(uid);
  resultsList.push(result);
});

// ===== 7. Flatten Results and Export to CSV =====
var allResults = ee.FeatureCollection(resultsList).flatten();

Export.table.toDrive({
  collection: allResults,
  description: 'ERA5Land_Selected_PV_Sites_2005_2024',
  folder: 'Meteorological_Analysis',
  fileFormat: 'CSV'
});

print('🚀 Batch processing for ERA5-Land initiated. Monitor progress in the Tasks tab.');
```
