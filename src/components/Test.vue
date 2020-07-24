<template>
  <div id="map"></div>
</template>

<script>
import 'ol/ol.css';
import { Map, View } from 'ol';
import { TileArcGISRest, XYZ } from 'ol/source';
import { Tile as TileLayer, Image as ImageLayer } from 'ol/layer';
import { get as getProjection, fromLonLat } from 'ol/proj';
import TileGrid from 'ol/tilegrid/TileGrid';

import { register } from 'ol/proj/proj4';
import proj4 from 'proj4';

proj4.defs('EPSG:4547', '+proj=tmerc +lat_0=0 +lon_0=114 +k=1 +x_0=500000 +y_0=0 +ellps=GRS80 +units=m +no_defs');
register(proj4);

let map = undefined;
let imgUrl = 'http://119.146.77.132:6080/arcgis/rest/services/' +
    '底图/2019年12月影像图/MapServer/tile/{z}/{y}/{x}';
let buildingUrl = 'http://172.16.217.62:6080/arcgis/rest/services/' +
    '建筑物普查/建筑物图斑/MapServer';

export default {
  methods: {
    
  },
  mounted() {
    let projection = getProjection('EPSG:4547');
    let origin = [ -5123200.0, 1.00021E7 ]
    let resolutions = [
      66.1459656252646,
      33.0729828126323,
      16.933367200067735,
      8.466683600033868,
      4.233341800016934,
      2.116670900008467,
      1.0583354500042335,
      0.5291677250021167,
      0.26458386250105836,
      0.13229193125052918
    ];
    let fullExtent = [ 511736.8282036524, 2534912.2933071232, 559191.97205, 2560733.6201400002 ];
    let tileGrid = new TileGrid({
      tileSize: 256,
      origin: origin,
      extent: fullExtent,
      resolutions: resolutions
    });
    let imageTileLayer = new TileLayer({
      source: new XYZ({
        url: imgUrl,
        projection: projection,
        tileGrid: tileGrid,
        wrapX: true
      })
    });
    let buildingLayer = new TileLayer({
      source: new TileArcGISRest({
        url: buildingUrl
      })
    });

    const mbToken = 'pk.eyJ1Ijoia29uZ3lsIiwiYSI6ImY1ZTk0NjY3ZjUyYzcwOTA4ZWE3ZWJmZTkwMjkzZjAzIn0.eAqcWKX1y_6xzNv3dkuiKA';
    let mbSource = new XYZ({
      url: 'https://api.mapbox.com/styles/v1/kongyl/ckahs7hty08gk1ipjtotpsn8e/tiles/256/{z}/{x}/{y}?access_token=' + mbToken
    });
    let mbLayer = new TileLayer({
      source: mbSource
    });

    map = new Map({
      target: 'map',
      layers: [
        mbLayer,
        imageTileLayer,
        buildingLayer
      ],
      view: new View({
        projection: 'EPSG:4547',
        center: fromLonLat([ 114.345661, 23.040161 ], 'EPSG:4547'),
        zoom: 13
      })
    });
  }
}
</script>

<style scoped>
#map {
  height: 100%;
}
</style>