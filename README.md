// =====================================================
// Trifinio – Temperatura 2m (°C) desde ERA5-Land HOURLY
// Agregación DIARIA y exportación POR AÑO (CSV)
// =====================================================

// 1) Puntos (lon, lat) del PDF
var ptGTM = ee.Feature(ee.Geometry.Point([-89.41065, 14.60314]),
  {country: 'Guatemala', lon: -89.41065, lat: 14.60314});
var ptHND = ee.Feature(ee.Geometry.Point([-89.11686, 14.48641]),
  {country: 'Honduras', lon: -89.11686, lat: 14.48641});
var ptSLV = ee.Feature(ee.Geometry.Point([-89.39434, 14.33633]),
  {country: 'El Salvador', lon: -89.39434, lat: 14.33633});
var trifinioPts = ee.FeatureCollection([ptGTM, ptHND, ptSLV]);

// (Opcional) buffer para promediar varios píxeles (~9 km de ERA5-Land)
var bufferedPts = trifinioPts.map(function(f){ return f.buffer(5000); });

// 2) Rango de años
var START_YEAR = 1981;
var END_YEAR   = 2024;

// 3) ERA5-Land horario (temperatura a 2 m en Kelvin)
var era5 = ee.ImageCollection('ECMWF/ERA5_LAND/HOURLY')
  .select('temperature_2m'); // K

// 4) Utilidades
function kToC(img) { return img.subtract(273.15); } // K → °C

// Construye colección diaria (tmax, tmin, tmean) para un año dado
function dailyTempsForYear(year) {
  var start = ee.Date.fromYMD(year, 1, 1);
  var end   = start.advance(1, 'year');
  var ic    = era5.filterDate(start, end);

  var nDays = end.difference(start, 'day');
  var dates = ee.List.sequence(0, nDays.subtract(1)).map(function(d){
    return start.advance(ee.Number(d), 'day');
  });

  var dailyIC = ee.ImageCollection.fromImages(
    dates.map(function(d){
      d = ee.Date(d);
      var next = d.advance(1, 'day');
      var dayIC = ic.filterDate(d, next); // 24 imágenes (horarias)

      var tmax  = kToC(dayIC.max()).rename('tmax_C');
      var tmin  = kToC(dayIC.min()).rename('tmin_C');
      var tmean = kToC(dayIC.mean()).rename('tmean_C');

      return ee.Image.cat([tmax, tmin, tmean])
        .set('date', d.format('YYYY-MM-dd'))
        .set('year', year);
    })
  );

  return dailyIC;
}

// Muestra una IC (diaria) en puntos → tabla
function sampleICtoTable(ic, points, scaleMeters) {
  var fc = ic.map(function(img){
    var dateStr = ee.String(img.get('date'));
    var sampled = img.reduceRegions({
      collection: points,
      reducer: ee.Reducer.mean(),
      scale: scaleMeters || 9000
    }).map(function(f){
      return ee.Feature(null, {
        country: f.get('country'),
        lon: f.get('lon'),
        lat: f.get('lat'),
        date: dateStr,
        tmax_C: f.get('tmax_C'),
        tmin_C: f.get('tmin_C'),
        tmean_C: f.get('tmean_C')
      });
    });
    return sampled;
  }).flatten();

  return ee.FeatureCollection(fc);
}

// 5) Bucle por año: crea una tarea de exportación por año
for (var year = START_YEAR; year <= END_YEAR; year++) {
  var icDaily   = dailyTempsForYear(year);
  var tableYear = sampleICtoTable(icDaily, bufferedPts, 9000);

  // Muestra pequeñita (evita prints enormes)
  print('Muestra diario ' + year, tableYear.limit(5));

  Export.table.toDrive({
    collection: tableYear,
    description: 'Trifinio_ERA5Land_temp_daily_' + year,
    fileNamePrefix: 'trifinio_era5land_temp_daily_' + year,
    fileFormat: 'CSV'
  });
}

// 6) Mapa (opcional y liviano)
Map.centerObject(trifinioPts, 8);
Map.addLayer(trifinioPts, {color: 'red'}, 'Puntos Trifinio');
