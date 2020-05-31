<template>
  <div id="map"></div>
</template>

<script>
import 'ol/ol.css';
import { Map, View, Overlay } from 'ol';
import { WMTS as WMTSSource, Vector as VectorSource } from 'ol/source';
import { get as getProjection, fromLonLat } from 'ol/proj';
import { getWidth, getTopLeft } from 'ol/extent';
import WMTSTileGrid from 'ol/tilegrid/WMTS';
import { Tile, Vector as VectorLayer } from 'ol/layer';
import FullScreen from 'ol/control/FullScreen';
import { Style, Fill, Stroke, Circle } from 'ol/style';
import { getLength, getArea } from 'ol/sphere';
import Draw from 'ol/interaction/Draw';
import { unByKey } from 'ol/Observable';

import 'ol-ext/control/Bar.css';
import Bar from 'ol-ext/control/Bar';
import Toggle from 'ol-ext/control/Toggle';

let map = undefined;

let measureSource = undefined;
let measureVector = undefined;
let measureTooltipElement = undefined;
let measureTooltip = undefined;
let sketch = undefined;
let listener = undefined;
let tooltipCoor = undefined;
let lengthDraw = undefined;
let areaDraw = undefined;

export default {
  methods: {
    initControl() {
      let mainbar = new Bar();
      mainbar.setPosition('top-left');
      map.addControl(mainbar);

      mainbar.addControl(new FullScreen());

      let measureTools = new Bar({
        toggleOne: true,
        group: true
      });
      mainbar.addControl(measureTools);
      this.initMeasure();
      let lengthTool = new Toggle({
        html: '<img src="/img/MeasureTool16.png">',
        title: '测量距离',
        interaction: lengthDraw,
        onToggle: function(active) {
          measureSource.clear();
          map.removeOverlay(measureTooltip);
          measureVector.setMap();
          if (active) {
            measureVector.setMap(map)
          }
        }
      });
      measureTools.addControl(lengthTool);
      let areaTool = new Toggle({
        html: '<img src="/img/MeasureAreaTool16.png">',
        title: '测量面积',
        interaction: areaDraw,
        onToggle: function(active) {
          measureSource.clear();
          map.removeOverlay(measureTooltip);
          measureVector.setMap();
          if (active) {
            measureVector.setMap(map)
          }
        }
      });
      measureTools.addControl(areaTool);
    },
    initMeasure() {
      measureSource = new VectorSource();
      let measureStyle = new Style({
        fill: new Fill({
          color: 'rgba(255, 255, 255, 0.2)'
        }),
        stroke: new Stroke({
          color: '#ffcc33',
          width: 2
        }),
        image: new Circle({
          radius: 7,
          fill: new Fill({
            color: '#ffcc33'
          })
        })
      });
      measureVector = new VectorLayer({
        source: measureSource, 
        style: measureStyle
      });

      let interMeasureStyle = new Style({
        fill: new Fill({
          color: 'rgba(255, 255, 255, 0.2)'
        }),
        stroke: new Stroke({
          color: 'rgba(0, 0, 0, 0.5)',
          lineDash: [ 10, 10 ],
          width: 2
        }),
        image: new Circle({
          radius: 5,
          stroke: new Stroke({
            color: 'rgba(0, 0, 0, 0.7)'
          }),
          fill: new Fill({
            color: 'rgba(255, 255, 255, 0.2)'
          })
        })
      });
      lengthDraw = new Draw({
        source: measureSource,
        type: 'LineString',
        style: interMeasureStyle
      });
      lengthDraw.on('drawstart', this.lengthDrawstart);
      lengthDraw.on('drawend', this.drawend);
      areaDraw = new Draw({
        source: measureSource,
        type: 'Polygon',
        style: interMeasureStyle
      });
      areaDraw.on('drawstart', this.areaDrawstart);
      areaDraw.on('drawend', this.drawend);
    },
    createMeasureTooltip() {
      if (measureTooltip) {
        map.removeOverlay(measureTooltip);
      }
      if (measureTooltipElement) {
        measureTooltipElement.parendNode.removeChild(measureTooltipElement);
      }
      measureTooltipElement = document.createElement('div');
      measureTooltipElement.className = 'ol-tooltip ol-tooltip-measure';
      measureTooltip = new Overlay({
        element: measureTooltipElement,
        offset: [ 0, -15 ],
        positioning: 'bottom-center'
      });
      map.addOverlay(measureTooltip);
    },
    formatLength(line) {
      let length = getLength(line);
      let output;
      if (length > 100) {
        output = (Math.round(length / 1000 * 100) / 100) + ' km';
      } else {
        output = (Math.round(length * 100) / 100) + ' m';
      }
      return output;
    },
    formatArea(polygon) {
      let area = getArea(polygon);
      let output;
      if (area > 10000) {
        output = (Math.round(area / 1000000 * 100) / 100) + ' km<sup>2</sup>';
      } else {
        output = (Math.round(area * 100) / 100) + ' m<sup>2</sup>';
      }
      return output;
    },
    lengthDrawstart(evt) {
      this.createMeasureTooltip();
      measureSource.clear();
      sketch = evt.feature;
      tooltipCoor = evt.coordinate;
      listener = sketch.getGeometry().on('change', this.lengthSketchChange);
    },
    areaDrawstart(evt) {
      this.createMeasureTooltip();
      measureSource.clear();
      sketch = evt.feature;
      tooltipCoor = evt.coordinate;
      listener = sketch.getGeometry().on('change', this.araeSketchChange);
    },
    drawend(evt) {
      measureTooltipElement.className = 'ol-tooltip ol-tooltip-static';
      measureTooltip.setOffset([ 0, -7 ]);
      sketch = null;
      measureTooltipElement = null;
      unByKey(listener);
    },
    lengthSketchChange(evt) {
      let geom = evt.target;
      let output = this.formatLength(geom);
      tooltipCoor = geom.getLastCoordinate();
      measureTooltipElement.innerHTML = output;
      measureTooltip.setPosition(tooltipCoor);
    },
    araeSketchChange(evt) {
      let geom = evt.target;
      let output = this.formatArea(geom);
      tooltipCoor = geom.getLastCoordinate();
      measureTooltipElement.innerHTML = output;
      measureTooltip.setPosition(tooltipCoor);
    }
  },
  mounted() {
    let projection = getProjection('EPSG:3857');
    let projectionExtent = projection.getExtent();
    let size = getWidth(projectionExtent) / 256;
    let resolutions = new Array(19);
    let tiandituMatrixIds = new Array(19);
    for (var z = 0; z < 19; z++) {
      resolutions[z] = size / Math.pow(2, z);
      tiandituMatrixIds[z] = z;
    }
    let tiandituTileGrid = new WMTSTileGrid({
      origin: getTopLeft(projectionExtent),
      resolutions: resolutions,
      matrixIds: tiandituMatrixIds
    });

    let tiandituVecLayer = new Tile({
      preload: Infinity,
      source: new WMTSSource({
        url: 'http://t{0-7}.tianditu.gov.cn/vec_w/wmts?tk=85001a47730b398597d5aea57512d84b',
        layer: 'vec',
        format: 'tiles',
        tileGrid: tiandituTileGrid,
        matrixSet: 'w',
        wrapX: true
      })
    });
    let tiandituCvaLayer = new Tile({
      preload: Infinity,
      source: new WMTSSource({
        url: 'http://t{0-7}.tianditu.gov.cn/cva_w/wmts?tk=85001a47730b398597d5aea57512d84b',
        layer: 'cva',
        format: 'tiles',
        tileGrid: tiandituTileGrid,
        matrixSet: 'w',
        wrapX: true
      })
    });

    map = new Map({
      target: 'map',
      layers: [
        tiandituVecLayer,
        tiandituCvaLayer
      ],
      view: new View({
        projection: projection,
        center: fromLonLat([ 114.345661, 23.040161 ]),
        zoom: 15
      })
    });

    this.initControl();
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#map {
  height: 100%;
}
</style>
