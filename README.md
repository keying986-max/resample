# resample
/***************************************************************
 * Annual Variables Auto-Resampling Template for Google Earth Engine
 *
 * Purpose:
 *   Generate a single-year aligned table from GEE datasets.
 *
 * Variables:
 *   - NDVI
 *   - NPP
 *   - Precipitation
 *   - ShortwaveRadiation
 *   - Temperature
 *   - Elevation
 *   - LandCover
 *   - SOC
 ***************************************************************/


/***************************************************************
 * 1. User configuration
 ***************************************************************/

var CONFIG = {
  // Replace this with your own study area boundary asset.
  // Example: 'users/your_username/your_study_area_asset'
  studyAreaAsset: 'users/your_username/your_study_area_asset',

  // Target year.
  targetYear: 2020,

  // Target output grid resolution in degrees.
  // Example:
  //   0.1  = approximately 10 km
  //   0.05 = approximately 5 km
  //   0.01 = approximately 1 km
  targetResolutionDeg: 0.1,

  // NDVI period.
  // Default: growing season mean NDVI from May to September.
  // For annual mean NDVI, set start month to 1 and end month to 12.
  ndviStartMonth: 5,
  ndviEndMonth: 9,

  // Export settings.
  exportFolder: 'GEE_Annual_Table',
  outputPrefix: 'annual_variables_resampled_table',
  tileScale: 8,

  // Approximate conversion from degrees to meters.
  metersPerDegree: 111320,

  // Fine-to-coarse aggregation settings.
  maxInputPixelsPerOutputPixel: 65535,
  bestEffort: true,

  // Similar-scale tolerance.
  scaleTolerance: 0.05,

  // Scale factors.
  ndviScaleFactor: 0.0001,

  // MOD17A3HGF Npp scale:
  // 0.0001 kg C/m2/year = 0.1 g C/m2/year
  nppScaleFactorToGramC: 0.1,

  // OpenLandMap SOC values are stored as x 5 g/kg.
  socScaleFactor: 5,

  // Optional vegetation mask.
  // If true, only pixels with NDVI > 0 and NPP > 0 will be exported.
  keepPositiveVegetationOnly: true,

  // Optional land-cover mask.
  // Keep empty if no land-cover class should be removed.
  // MODIS IGBP examples:
  //   13 = Urban and built-up lands
  //   15 = Snow and ice
  //   17 = Water bodies
  excludedLandCoverValues: [],

  // Map display.
  mapZoom: 6
};


/***************************************************************
 * 2. GEE dataset IDs
 ***************************************************************/

var DATASETS = {
  ndvi: 'MODIS/061/MOD13A1',
  npp: 'MODIS/061/MOD17A3HGF',
  climate: 'ECMWF/ERA5_LAND/DAILY_AGGR',
  elevation: 'USGS/SRTMGL1_003',
  landCover: 'MODIS/061/MCD12Q1',
  soilCarbon: 'OpenLandMap/SOL/SOL_ORGANIC-CARBON_USDA-6A1C_M/v02'
};


/***************************************************************
 * 3. Helper functions
 ***************************************************************/

function makeWgs84Projection(resolutionDeg) {
  return ee.Projection('EPSG:4326', [
    resolutionDeg, 0, -180,
    0, -resolutionDeg, 90
  ]);
}

function cleanName(text) {
  return String(text).replace(/[^A-Za-z0-9_]/g, '_');
}

function getDateRange(year) {
  var startDate = ee.Date.fromYMD(year, 1, 1);
  var endDate = startDate.advance(1, 'year');

  return {
    start: startDate,
    end: endDate
  };
}

function getNdviDateRange(year) {
  var startDate = ee.Date.fromYMD(
    year,
    CONFIG.ndviStartMonth,
    1
  );

  var endDate = ee.Date.fromYMD(
    year,
    CONFIG.ndviEndMonth,
    1
  ).advance(1, 'month');

  return {
    start: startDate,
    end: endDate
  };
}

function getReducer(method) {
  if (method === 'mean') {
    return ee.Reducer.mean();
  }

  if (method === 'median') {
    return ee.Reducer.median();
  }

  if (method === 'sum') {
    return ee.Reducer.sum();
  }

  if (method === 'min') {
    return ee.Reducer.min();
  }

  if (method === 'max') {
    return ee.Reducer.max();
  }

  if (method === 'mode') {
    return ee.Reducer.mode();
  }

  throw new Error('Unsupported reducer: ' + method);
}

function stackImageArray(imageArray) {
  var stacked = ee.Image(imageArray[0]);

  for (var i = 1; i < imageArray.length; i++) {
    stacked = stacked.addBands(ee.Image(imageArray[i]));
  }

  return stacked;
}

function validateConfig(config) {
  var placeholder = 'users/your_username/your_study_area_asset';

  if (!config.studyAreaAsset || config.studyAreaAsset === placeholder) {
    throw new Error(
      'Please replace CONFIG.studyAreaAsset with your own study area boundary asset.'
    );
  }

  if (typeof config.targetYear !== 'number') {
    throw new Error('CONFIG.targetYear should be a number.');
  }

  if (config.targetResolutionDeg <= 0) {
    throw new Error('CONFIG.targetResolutionDeg should be greater than 0.');
  }

  if (config.ndviStartMonth < 1 || config.ndviEndMonth > 12) {
    throw new Error('NDVI month settings should be between 1 and 12.');
  }

  if (config.ndviStartMonth > config.ndviEndMonth) {
    throw new Error('CONFIG.ndviStartMonth should not be greater than CONFIG.ndviEndMonth.');
  }
}

validateConfig(CONFIG);


/***************************************************************
 * 4. Build annual variables
 ***************************************************************/

/**
 * NDVI from MOD13A1.
 *
 * Temporal operation:
 *   Mean NDVI during the selected months.
 *
 * Default:
 *   Growing season mean NDVI from May to September.
 */
function getAnnualNdvi(year) {
  var dates = getNdviDateRange(year);

  var collection = ee.ImageCollection(DATASETS.ndvi)
    .filterDate(dates.start, dates.end);

  var referenceProjection = ee.Image(collection.first())
    .select('NDVI')
    .projection();

  var ndviCollection = collection.map(function(image) {
    var ndvi = image
      .select('NDVI')
      .multiply(CONFIG.ndviScaleFactor)
      .rename('NDVI');

    var qa = image.select('SummaryQA');

    // SummaryQA:
    // 0 = good data
    // 1 = marginal data
    var mask = qa.lte(1);

    return ndvi.updateMask(mask);
  });

  return ndviCollection
    .mean()
    .rename('NDVI')
    .setDefaultProjection(referenceProjection);
}


/**
 * Annual NPP from MOD17A3HGF.
 *
 * Output unit:
 *   g C/m2/year
 */
function getAnnualNpp(year) {
  var dates = getDateRange(year);

  var raw = ee.Image(
    ee.ImageCollection(DATASETS.npp)
      .filterDate(dates.start, dates.end)
      .first()
  );

  var referenceProjection = raw
    .select('Npp')
    .projection();

  return raw
    .select('Npp')
    .multiply(CONFIG.nppScaleFactorToGramC)
    .rename('NPP')
    .setDefaultProjection(referenceProjection);
}


/**
 * Annual precipitation from ERA5-Land Daily Aggregated.
 *
 * Temporal operation:
 *   Sum of daily precipitation.
 *
 * Output unit:
 *   mm/year
 */
function getAnnualPrecipitation(year) {
  var dates = getDateRange(year);

  var collection = ee.ImageCollection(DATASETS.climate)
    .filterDate(dates.start, dates.end);

  var referenceProjection = ee.Image(collection.first())
    .select('total_precipitation_sum')
    .projection();

  return collection
    .select('total_precipitation_sum')
    .sum()
    .multiply(1000)
    .rename('Precipitation')
    .setDefaultProjection(referenceProjection);
}


/**
 * Annual shortwave radiation from ERA5-Land Daily Aggregated.
 *
 * Temporal operation:
 *   Sum of daily shortwave radiation.
 *
 * Output unit:
 *   J/m2/year
 */
function getAnnualShortwaveRadiation(year) {
  var dates = getDateRange(year);

  var collection = ee.ImageCollection(DATASETS.climate)
    .filterDate(dates.start, dates.end);

  var referenceProjection = ee.Image(collection.first())
    .select('surface_solar_radiation_downwards_sum')
    .projection();

  return collection
    .select('surface_solar_radiation_downwards_sum')
    .sum()
    .rename('ShortwaveRadiation')
    .setDefaultProjection(referenceProjection);
}


/**
 * Annual mean near-surface air temperature from ERA5-Land Daily Aggregated.
 *
 * Temporal operation:
 *   Mean of daily 2 m air temperature.
 *
 * Output unit:
 *   degree Celsius
 */
function getAnnualTemperature(year) {
  var dates = getDateRange(year);

  var collection = ee.ImageCollection(DATASETS.climate)
    .filterDate(dates.start, dates.end);

  var referenceProjection = ee.Image(collection.first())
    .select('temperature_2m')
    .projection();

  return collection
    .select('temperature_2m')
    .mean()
    .subtract(273.15)
    .rename('Temperature')
    .setDefaultProjection(referenceProjection);
}


/**
 * Elevation from SRTM.
 *
 * Output unit:
 *   m
 */
function getElevation() {
  var raw = ee.Image(DATASETS.elevation)
    .select('elevation');

  return raw
    .rename('Elevation')
    .setDefaultProjection(raw.projection());
}


/**
 * Annual land cover from MODIS MCD12Q1.
 *
 * Band:
 *   LC_Type1
 */
function getAnnualLandCover(year) {
  var dates = getDateRange(year);

  var raw = ee.Image(
    ee.ImageCollection(DATASETS.landCover)
      .filterDate(dates.start, dates.end)
      .first()
  );

  var referenceProjection = raw
    .select('LC_Type1')
    .projection();

  return raw
    .select('LC_Type1')
    .rename('LandCover')
    .setDefaultProjection(referenceProjection);
}


/**
 * Soil organic carbon from OpenLandMap.
 *
 * Depth treatment:
 *   Mean of 0, 10, and 30 cm layers.
 *
 * Output unit:
 *   g/kg
 */
function getSoc0To30cm() {
  var raw = ee.Image(DATASETS.soilCarbon)
    .select(['b0', 'b10', 'b30'])
    .multiply(CONFIG.socScaleFactor);

  return raw
    .reduce(ee.Reducer.mean())
    .rename('SOC')
    .setDefaultProjection(raw.projection());
}


/***************************************************************
 * 5. Basic variables
 ***************************************************************/

var studyArea = ee.FeatureCollection(CONFIG.studyAreaAsset).geometry();

var targetProjection = makeWgs84Projection(CONFIG.targetResolutionDeg);

var targetScaleMeters = CONFIG.targetResolutionDeg * CONFIG.metersPerDegree;

var yearTag = String(CONFIG.targetYear);

print('Target year:', CONFIG.targetYear);
print('NDVI period:', CONFIG.ndviStartMonth + '-' + CONFIG.ndviEndMonth);
print('Target resolution degree:', CONFIG.targetResolutionDeg);
print('Approximate target scale in meters:', targetScaleMeters);
print('Target projection:', targetProjection);


/***************************************************************
 * 6. Source image list
 *
 * Resampling rules:
 *   Continuous variables:
 *     fine -> coarse: mean
 *     coarse -> fine: bilinear
 *
 *   Categorical variables:
 *     fine -> coarse: mode
 *     coarse -> fine: nearest
 ***************************************************************/

var sourceItems = [
  {
    name: 'NDVI',
    image: getAnnualNdvi(CONFIG.targetYear),
    nativeScaleMeters: 500,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'NPP',
    image: getAnnualNpp(CONFIG.targetYear),
    nativeScaleMeters: 500,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'Precipitation',
    image: getAnnualPrecipitation(CONFIG.targetYear),
    nativeScaleMeters: 11132,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'ShortwaveRadiation',
    image: getAnnualShortwaveRadiation(CONFIG.targetYear),
    nativeScaleMeters: 11132,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'Temperature',
    image: getAnnualTemperature(CONFIG.targetYear),
    nativeScaleMeters: 11132,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'Elevation',
    image: getElevation(),
    nativeScaleMeters: 30,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'LandCover',
    image: getAnnualLandCover(CONFIG.targetYear),
    nativeScaleMeters: 500,
    dataType: 'categorical',
    aggregationReducer: 'mode',
    interpolationMethod: 'nearest'
  },
  {
    name: 'SOC',
    image: getSoc0To30cm(),
    nativeScaleMeters: 250,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  }
];


/***************************************************************
 * 7. Automatic resampling
 ***************************************************************/

function getResamplingAction(item) {
  var lowerBound = targetScaleMeters * (1 - CONFIG.scaleTolerance);
  var upperBound = targetScaleMeters * (1 + CONFIG.scaleTolerance);

  if (item.nativeScaleMeters < lowerBound) {
    return 'fine_to_coarse';
  }

  if (item.nativeScaleMeters > upperBound) {
    return 'coarse_to_fine';
  }

  return 'same_or_similar';
}


function resampleToTargetGrid(item) {
  var image = ee.Image(item.image)
    .rename(item.name);

  var action = getResamplingAction(item);

  print(item.name + ' native scale in meters:', item.nativeScaleMeters);
  print(item.name + ' resampling action:', action);

  if (action === 'fine_to_coarse') {
    var reducer;

    if (item.dataType === 'categorical') {
      reducer = ee.Reducer.mode();
    } else {
      reducer = getReducer(item.aggregationReducer);
    }

    return image
      .reduceResolution({
        reducer: reducer,
        maxPixels: CONFIG.maxInputPixelsPerOutputPixel,
        bestEffort: CONFIG.bestEffort
      })
      .reproject({
        crs: targetProjection
      })
      .rename(item.name);
  }

  if (action === 'coarse_to_fine') {
    if (item.dataType === 'categorical') {
      return image
        .reproject({
          crs: targetProjection
        })
        .rename(item.name);
    }

    return image
      .resample(item.interpolationMethod)
      .reproject({
        crs: targetProjection
      })
      .rename(item.name);
  }

  return image
    .reproject({
      crs: targetProjection
    })
    .rename(item.name);
}


var resampledImages = sourceItems.map(resampleToTargetGrid);

var stacked = stackImageArray(resampledImages)
  .clip(studyArea);

print('Stacked image bands:', stacked.bandNames());


/***************************************************************
 * 8. Optional masks
 ***************************************************************/

if (CONFIG.keepPositiveVegetationOnly) {
  var validVegetationMask = stacked
    .select('NDVI')
    .gt(0)
    .and(stacked.select('NPP').gt(0));

  stacked = stacked.updateMask(validVegetationMask);
}

if (CONFIG.excludedLandCoverValues.length > 0) {
  var landCoverMask = ee.Image(1);

  CONFIG.excludedLandCoverValues.forEach(function(value) {
    landCoverMask = landCoverMask.and(
      stacked.select('LandCover').neq(value)
    );
  });

  stacked = stacked.updateMask(landCoverMask);
}


/***************************************************************
 * 9. Add longitude and latitude
 ***************************************************************/

var lonLat = ee.Image.pixelLonLat()
  .reproject({
    crs: targetProjection
  })
  .rename(['lon', 'lat']);

var outputImage = stacked.addBands(lonLat);


/***************************************************************
 * 10. Sample aligned table
 ***************************************************************/

var variableNames = [
  'NDVI',
  'NPP',
  'Precipitation',
  'ShortwaveRadiation',
  'Temperature',
  'Elevation',
  'LandCover',
  'SOC'
];

var selectors = ['lon', 'lat', 'year'].concat(variableNames);

var requiredFields = ['lon', 'lat'].concat(variableNames);

var samples = outputImage.sample({
  region: studyArea,
  projection: targetProjection,
  geometries: false,
  dropNulls: false,
  tileScale: CONFIG.tileScale
}).map(function(feature) {
  return feature.set('year', CONFIG.targetYear);
});

samples = samples.filter(ee.Filter.notNull(requiredFields));

print('Sample count after removing missing rows:', samples.size());
print('First sample:', samples.first());


/***************************************************************
 * 11. Visualization
 ***************************************************************/

Map.centerObject(studyArea, CONFIG.mapZoom);

Map.addLayer(
  stacked.select('NDVI'),
  {
    min: 0,
    max: 0.8
  },
  'NDVI'
);

Map.addLayer(
  stacked.select('NPP'),
  {
    min: 0,
    max: 600
  },
  'NPP'
);

Map.addLayer(
  stacked.select('Precipitation'),
  {
    min: 0,
    max: 1000
  },
  'Precipitation'
);

Map.addLayer(
  stacked.select('ShortwaveRadiation'),
  {
    min: 0,
    max: 9000000000
  },
  'Shortwave Radiation'
);

Map.addLayer(
  stacked.select('Temperature'),
  {
    min: -10,
    max: 30
  },
  'Temperature'
);

Map.addLayer(
  stacked.select('Elevation'),
  {
    min: 0,
    max: 6000
  },
  'Elevation'
);

Map.addLayer(
  stacked.select('LandCover'),
  {
    min: 1,
    max: 17
  },
  'Land Cover'
);

Map.addLayer(
  stacked.select('SOC'),
  {
    min: 0,
    max: 150
  },
  'SOC'
);


/***************************************************************
 * 12. Export table
 ***************************************************************/

Export.table.toDrive({
  collection: samples,
  description: cleanName(CONFIG.outputPrefix + '_' + yearTag),
  folder: CONFIG.exportFolder,
  fileNamePrefix: cleanName(CONFIG.outputPrefix + '_' + yearTag),
  fileFormat: 'CSV',
  selectors: selectors
});

print('Export task has been created. Please run it in the Tasks panel.');
