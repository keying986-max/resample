 * 1. User configuration
 ******************************************************************************/

var CONFIG = {
  // Script-level metadata for reproducible exports.
  scriptVersion: '1.0.0',
  analysisId: 'annual_variables_v1',

  // Replace this placeholder with your own study area boundary asset.
  // Example: 'users/your_username/your_study_area_asset'
  studyAreaAsset: 'users/your_username/your_study_area_asset',

  // Target year for annual variables.
  targetYear: 2020,

  // Target grid settings.
  // The affine transform below anchors the grid globally at (-180, 90).
  targetCrs: 'EPSG:4326',
  targetResolutionDeg: 0.1,
  targetGridOriginLon: -180,
  targetGridOriginLat: 90,

  // Approximate conversion from degrees to meters.
  // Used only for deciding whether a source dataset is finer or coarser than
  // the target grid. Reprojection itself uses the explicit target projection.
  metersPerDegree: 111320,

  // NDVI temporal window.
  // Default: growing-season mean NDVI from May to September.
  // For annual mean NDVI, set ndviStartMonth = 1 and ndviEndMonth = 12.
  ndviStartMonth: 5,
  ndviEndMonth: 9,

  // Export settings.
  exportFolder: 'GEE_Annual_Table',
  outputPrefix: 'annual_variables_resampled_table',
  tileScale: 8,

  // Fine-to-coarse aggregation settings.
  maxInputPixelsPerOutputPixel: 65535,
  bestEffort: true,

  // Scale-comparison tolerance used by getResamplingAction().
  // Example: 0.05 means that native and target scales within +/-5% are treated
  // as equivalent.
  scaleTolerance: 0.05,

  // Dataset-specific scale factors.
  ndviScaleFactor: 0.0001,

  // MOD17A3HGF Npp scale:
  // 0.0001 kg C m-2 yr-1 = 0.1 g C m-2 yr-1.
  nppScaleFactorToGramC: 0.1,

  // OpenLandMap SOC values are stored as x 5 g kg-1.
  socScaleFactor: 5,

  // Optional vegetation mask.
  // If true, only pixels with NDVI > 0 and NPP > 0 are exported.
  keepPositiveVegetationOnly: true,

  // Optional land-cover mask.
  // Keep this array empty if no land-cover class should be removed.
  // MODIS IGBP examples:
  //   13 = Urban and built-up lands
  //   15 = Snow and ice
  //   17 = Water bodies
  excludedLandCoverValues: [],

  // Map display.
  mapZoom: 6
};


/******************************************************************************
 * 2. GEE dataset IDs
 ******************************************************************************/

var DATASETS = {
  ndvi: 'MODIS/061/MOD13A1',
  npp: 'MODIS/061/MOD17A3HGF',
  climate: 'ECMWF/ERA5_LAND/DAILY_AGGR',
  elevation: 'USGS/SRTMGL1_003',
  landCover: 'MODIS/061/MCD12Q1',
  soilCarbon: 'OpenLandMap/SOL/SOL_ORGANIC-CARBON_USDA-6A1C_M/v02'
};


/******************************************************************************
 * 3. Variable definitions
 *
 * nativeScaleMeters is used for resampling decisions only. It is intentionally
 * specified here instead of inferred dynamically so that the resampling logic is
 * transparent and reproducible.
 ******************************************************************************/

var VARIABLE_SPECS = [
  {
    name: 'NDVI',
    nativeScaleMeters: 500,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'NPP',
    nativeScaleMeters: 500,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'Precipitation',
    nativeScaleMeters: 11132,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'ShortwaveRadiation',
    nativeScaleMeters: 11132,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'Temperature',
    nativeScaleMeters: 11132,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'Elevation',
    nativeScaleMeters: 30,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  },
  {
    name: 'LandCover',
    nativeScaleMeters: 500,
    dataType: 'categorical',
    aggregationReducer: 'mode',
    interpolationMethod: 'nearest'
  },
  {
    name: 'SOC',
    nativeScaleMeters: 250,
    dataType: 'continuous',
    aggregationReducer: 'mean',
    interpolationMethod: 'bilinear'
  }
];


/******************************************************************************
 * 4. Helper functions
 ******************************************************************************/

function makeWgs84Projection(config) {
  return ee.Projection(config.targetCrs, [
    config.targetResolutionDeg,
    0,
    config.targetGridOriginLon,
    0,
    -config.targetResolutionDeg,
    config.targetGridOriginLat
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
  if (imageArray.length === 0) {
    throw new Error('The image array is empty. At least one image is required.');
  }

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

  if (config.targetYear < 2000 || config.targetYear > 2100) {
    throw new Error('CONFIG.targetYear is outside the expected range: 2000-2100.');
  }

  if (config.targetCrs !== 'EPSG:4326') {
    throw new Error('This template currently expects CONFIG.targetCrs to be EPSG:4326.');
  }

  if (config.targetResolutionDeg <= 0) {
    throw new Error('CONFIG.targetResolutionDeg should be greater than 0.');
  }

  if (config.ndviStartMonth < 1 || config.ndviStartMonth > 12) {
    throw new Error('CONFIG.ndviStartMonth should be between 1 and 12.');
  }

  if (config.ndviEndMonth < 1 || config.ndviEndMonth > 12) {
    throw new Error('CONFIG.ndviEndMonth should be between 1 and 12.');
  }

  if (config.ndviStartMonth > config.ndviEndMonth) {
    throw new Error(
      'CONFIG.ndviStartMonth should not be greater than CONFIG.ndviEndMonth.'
    );
  }

  if (config.scaleTolerance < 0 || config.scaleTolerance >= 1) {
    throw new Error('CONFIG.scaleTolerance should be in the interval [0, 1).');
  }
}

function copySpecWithImage(spec, image) {
  var output = {};
  var keys = Object.keys(spec);

  keys.forEach(function(key) {
    output[key] = spec[key];
  });

  output.image = image;
  return output;
}

function addRunMetadata(feature) {
  return feature.set({
    analysis_id: CONFIG.analysisId,
    script_version: CONFIG.scriptVersion,
    year: CONFIG.targetYear,
    target_crs: CONFIG.targetCrs,
    target_resolution_deg: CONFIG.targetResolutionDeg,
    ndvi_start_month: CONFIG.ndviStartMonth,
    ndvi_end_month: CONFIG.ndviEndMonth,
    keep_positive_vegetation_only: CONFIG.keepPositiveVegetationOnly,
    excluded_land_cover_values: CONFIG.excludedLandCoverValues.join('|')
  });
}

validateConfig(CONFIG);


/******************************************************************************
 * 5. Build annual variables
 ******************************************************************************/

/**
 * NDVI from MOD13A1.
 *
 * Temporal operation:
 *   Mean NDVI during the selected months.
 *
 * Quality control:
 *   SummaryQA <= 1, where 0 = good data and 1 = marginal data.
 *
 * Output unit:
 *   Unitless NDVI.
 */
function getAnnualNdvi(year) {
  var dates = getNdviDateRange(year);

  var collection = ee.ImageCollection(DATASETS.ndvi)
    .filterDate(dates.start, dates.end);

  print('NDVI image count:', collection.size());

  var referenceProjection = ee.Image(collection.first())
    .select('NDVI')
    .projection();

  var ndviCollection = collection.map(function(image) {
    var ndvi = image
      .select('NDVI')
      .multiply(CONFIG.ndviScaleFactor)
      .rename('NDVI');

    var qa = image.select('SummaryQA');
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
 *   g C m-2 yr-1.
 */
function getAnnualNpp(year) {
  var dates = getDateRange(year);

  var collection = ee.ImageCollection(DATASETS.npp)
    .filterDate(dates.start, dates.end);

  print('NPP image count:', collection.size());

  var raw = ee.Image(collection.first());

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
 *   mm yr-1.
 */
function getAnnualPrecipitation(year) {
  var dates = getDateRange(year);

  var collection = ee.ImageCollection(DATASETS.climate)
    .filterDate(dates.start, dates.end);

  print('ERA5-Land daily image count for precipitation:', collection.size());

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
 *   J m-2 yr-1.
 */
function getAnnualShortwaveRadiation(year) {
  var dates = getDateRange(year);

  var collection = ee.ImageCollection(DATASETS.climate)
    .filterDate(dates.start, dates.end);

  print('ERA5-Land daily image count for shortwave radiation:', collection.size());

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
 *   degree Celsius.
 */
function getAnnualTemperature(year) {
  var dates = getDateRange(year);

  var collection = ee.ImageCollection(DATASETS.climate)
    .filterDate(dates.start, dates.end);

  print('ERA5-Land daily image count for temperature:', collection.size());

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
 *   m.
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
 *   LC_Type1, IGBP classification.
 */
function getAnnualLandCover(year) {
  var dates = getDateRange(year);

  var collection = ee.ImageCollection(DATASETS.landCover)
    .filterDate(dates.start, dates.end);

  print('Land-cover image count:', collection.size());

  var raw = ee.Image(collection.first());

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
 *   g kg-1.
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


/******************************************************************************
 * 6. Study area and target grid
 ******************************************************************************/

var studyArea = ee.FeatureCollection(CONFIG.studyAreaAsset).geometry();

var targetProjection = makeWgs84Projection(CONFIG);

var targetScaleMeters = CONFIG.targetResolutionDeg * CONFIG.metersPerDegree;

var yearTag = String(CONFIG.targetYear);
var exportName = cleanName(CONFIG.outputPrefix + '_' + yearTag);

print('Run metadata:', CONFIG);
print('Datasets:', DATASETS);
print('Target projection:', targetProjection);
print('Approximate target scale in meters:', targetScaleMeters);


/******************************************************************************
 * 7. Build source image list
 ******************************************************************************/

var SOURCE_IMAGES_BY_NAME = {
  NDVI: getAnnualNdvi(CONFIG.targetYear),
  NPP: getAnnualNpp(CONFIG.targetYear),
  Precipitation: getAnnualPrecipitation(CONFIG.targetYear),
  ShortwaveRadiation: getAnnualShortwaveRadiation(CONFIG.targetYear),
  Temperature: getAnnualTemperature(CONFIG.targetYear),
  Elevation: getElevation(),
  LandCover: getAnnualLandCover(CONFIG.targetYear),
  SOC: getSoc0To30cm()
};

var sourceItems = VARIABLE_SPECS.map(function(spec) {
  return copySpecWithImage(spec, SOURCE_IMAGES_BY_NAME[spec.name]);
});


/******************************************************************************
 * 8. Automatic resampling
 *
 * Resampling rules
 * ----------------
 * Continuous variables:
 *   - fine -> coarse: reducer specified by aggregationReducer, usually mean
 *   - coarse -> fine: interpolationMethod, usually bilinear
 *
 * Categorical variables:
 *   - fine -> coarse: mode
 *   - coarse -> fine: nearest-neighbor behavior through reprojection
 ******************************************************************************/

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
  var image = ee.Image(item.image).rename(item.name);
  var action = getResamplingAction(item);

  print(item.name + ' native scale in meters:', item.nativeScaleMeters);
  print(item.name + ' resampling action:', action);

  if (action === 'fine_to_coarse') {
    var reducer = item.dataType === 'categorical'
      ? ee.Reducer.mode()
      : getReducer(item.aggregationReducer);

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


/******************************************************************************
 * 9. Optional masks
 ******************************************************************************/

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


/******************************************************************************
 * 10. Add longitude and latitude
 ******************************************************************************/

var lonLat = ee.Image.pixelLonLat()
  .reproject({
    crs: targetProjection
  })
  .rename(['lon', 'lat']);

var outputImage = stacked.addBands(lonLat);


/******************************************************************************
 * 11. Sample aligned table
 ******************************************************************************/

var variableNames = VARIABLE_SPECS.map(function(spec) {
  return spec.name;
});

var metadataFields = [
  'analysis_id',
  'script_version',
  'year',
  'target_crs',
  'target_resolution_deg',
  'ndvi_start_month',
  'ndvi_end_month',
  'keep_positive_vegetation_only',
  'excluded_land_cover_values'
];

var selectors = ['lon', 'lat'].concat(metadataFields).concat(variableNames);
var requiredFields = ['lon', 'lat'].concat(variableNames);

var samples = outputImage.sample({
  region: studyArea,
  projection: targetProjection,
  geometries: false,
  dropNulls: false,
  tileScale: CONFIG.tileScale
}).map(addRunMetadata);

samples = samples.filter(ee.Filter.notNull(requiredFields));

print('Sample count after removing missing rows:', samples.size());
print('First sample:', samples.first());


/******************************************************************************
 * 12. Visualization
 ******************************************************************************/

Map.centerObject(studyArea, CONFIG.mapZoom);

Map.addLayer(
  stacked.select('NDVI'),
  {min: 0, max: 0.8},
  'NDVI'
);

Map.addLayer(
  stacked.select('NPP'),
  {min: 0, max: 600},
  'NPP'
);

Map.addLayer(
  stacked.select('Precipitation'),
  {min: 0, max: 1000},
  'Precipitation'
);

Map.addLayer(
  stacked.select('ShortwaveRadiation'),
  {min: 0, max: 9000000000},
  'Shortwave Radiation'
);

Map.addLayer(
  stacked.select('Temperature'),
  {min: -10, max: 30},
  'Temperature'
);

Map.addLayer(
  stacked.select('Elevation'),
  {min: 0, max: 6000},
  'Elevation'
);

Map.addLayer(
  stacked.select('LandCover'),
  {min: 1, max: 17},
  'Land Cover'
);

Map.addLayer(
  stacked.select('SOC'),
  {min: 0, max: 150},
  'SOC'
);


/******************************************************************************
 * 13. Export table
 ******************************************************************************/

Export.table.toDrive({
  collection: samples,
  description: exportName,
  folder: CONFIG.exportFolder,
  fileNamePrefix: exportName,
  fileFormat: 'CSV',
  selectors: selectors
});

print('Export task has been created. Please run it in the Tasks panel.');
print('Export file name prefix:', exportName);
