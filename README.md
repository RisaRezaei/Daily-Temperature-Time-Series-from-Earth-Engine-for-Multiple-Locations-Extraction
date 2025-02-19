# Daily Temperature Time Series from Earth Engine for Multiple Locations Extraction

## Overview

This script processes daily temperature time series using Google Earth Engine. It extracts temperature data from an image collection, averages it over daily intervals, and associates it with specific station locations. The results are formatted into a structured table, merging overlapping observations to retain the maximum value for each day. Finally, the processed data is exported as a CSV file for further analysis.

---
## Code Explanation

### **Step 1: Load Feature Collection**
Load the feature collection of points from the variable `st`.
```javascript
var points = ee.FeatureCollection(st);
```

### **Step 2: Define a Function to Aggregate Data by a Time Interval**
`daily_product`: Aggregates an image collection by a given interval.

 * Inputs:

`collection`: The image collection to be aggregated.

`start`: The start date for aggregation.

`count`: Number of intervals to process.

`interval`: Size of the time step.

`units`: Time unit (e.g., 'day').

 * Output:
 
 `ImageCollection`: having images averaged over each time interval.
 
```javascript
var daily_product = function(collection, start, count, interval, units) {
  var sequence = ee.List.sequence(0, ee.Number(count).subtract(1));
  var origin_date = ee.Date(start);
  
  return ee.ImageCollection(sequence.map(function(i) {
    var start_date = origin_date.advance(ee.Number(interval).multiply(i), units);
    var end_date = origin_date.advance(ee.Number(interval).multiply(ee.Number(i).add(1)), units);
    
    return collection.filterDate(start_date, end_date).mean()
      .set('system:time_start', start_date.millis())
      .set('system:time_end', end_date.millis());
  }));
};

```
### **Step 3: Extract Relevant Data from Images**
* `DATA`: Extracts 'temperature_2m' band and clips it to the region of interest.
 * Preserves metadata properties.

```javascript
var DATA = function(img) {
  var bands = img.select('temperature_2m').clip(st);
  return bands.copyProperties(img, ['system:time_start', 'system:time_end']);
};
```

### **Step 4: Filter and Process the Image Collection**
 * Filter images by date and region.
 * Map `DATA` function over the collection.

```javascript
var Tair = imageCollection
  .filterDate('1986-09-23', '2018-09-23')
  .filterBounds(geometry)
  .map(DATA);
```

### **Step 5: Generate Daily Temperature Data**
 * Apply `daily_product` to create a daily time series (11690 days).
```javascript
var T_daily = daily_product(Tair, '1986-09-23', 11690, 1, 'day');
var collection = T_daily;
print(collection);
```

### **Step 6: Extract Mean Temperature at Station Locations**
 * Map function over the collection to compute mean temperature at each station.
```javascript
var triplets = collection.map(function(image) {
  return image.select('temperature_2m').reduceRegions({
    collection: points,
    reducer: ee.Reducer.mean().setOutputs(['temperature_2m']),
    scale: 11132,
  }).map(function(feature) {
    var ii = ee.Date(image.get('system:time_start'));
    var i = ii.format("YY-MM-dd");
    return feature.set({'imageID': i});
  });
}).flatten();
```

### **Step 7: Format Data into Station-wise Time Series**
 * `format`: Converts data into a structured table format.
 * Groups by station ID, with temperature values indexed by date.
```javascript
var format = function(table, rowId, colId) {
  var rows = table.distinct(rowId);
  var joined = ee.Join.saveAll('matches').apply({
    primary: rows,
    secondary: table,
    condition: ee.Filter.equals({ leftField: rowId, rightField: rowId })
  });
  
  return joined.map(function(row) {
    var values = ee.List(row.get('matches'))
      .map(function(feature) {
        feature = ee.Feature(feature);
        return [feature.get(colId), feature.get('temperature_2m')];
      });
    return row.select([rowId]).set(ee.Dictionary(values.flatten()));
  });
};
var sentinelResults = format(triplets, 'stationid', 'imageID');
print(triplets.first());
print(sentinelResults.first());
```

### **Step 8: Merge Overlapping Observations**
 * `merge`: Aggregates temperature values when multiple records exist for a date.
 * Uses maximum temperature value among duplicates.
```javascript
var merge = function(table, rowId) {
  return table.map(function(feature) {
    var id = feature.get(rowId);
    var allKeys = feature.toDictionary().keys().remove(rowId);
    var substrKeys = ee.List(allKeys.map(function(val) {
      return ee.String(val).slice(0, 8);
    }));
    var uniqueKeys = substrKeys.distinct();
    var pairs = uniqueKeys.map(function(key) {
      var matches = feature.toDictionary().select(allKeys.filter(ee.Filter.stringContains('item', key))).values();
      var val = matches.reduce(ee.Reducer.max());
      return [key, val];
    });
    return feature.select([rowId]).set(ee.Dictionary(pairs.flatten()));
  });
};
var sentinelMerged = merge(sentinelResults, 'stationid');
```
### **Step 9: Export the Processed Data to Google Drive**
 * Saves data in CSV format under the 'earthengine' folder.
```javascript
Export.table.toDrive({
  collection: sentinelMerged,
  description: 'Multiple_Locations_T_time_series',
  folder: 'earthengine',
  fileNamePrefix: 'T_time_series_multiple',
  fileFormat: 'CSV'
});
```
