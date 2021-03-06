<template>
  <MglMap
    :center="mapCenter"
    :zoom="mapZoom"
    :map-style="mapStyle"
    :attribution-control="false"
    @load="load"
    @update:center="updateMapCenter"
    @update:zoom="updateMapZoom"
  >
    <MglNavigationControl :show-compass="false" />
    <MglGeolocateControl
      :positionOptions="{ enableHighAccuracy: true }"
    />
    <MglAttributionControl
      :compact="false"
      position="bottom-right"
    />
    <MglVectorLayer
      v-for="layer in layers"
      :key="layer.id"
      :clear-source="false"
      :layer-id="layer.id"
      :layer="layer"
      :source="poiSource"
      source-id="poi"
      @mouseenter="mouseenter"
      @click="clickPoi"
      @mouseleave="mouseleave"
    />
  </MglMap>
</template>

<script>
import * as config from '../config.json';
import { MglMap, MglNavigationControl, MglVectorLayer, MglAttributionControl, MglGeolocateControl } from 'vue-mapbox/dist/vue-mapbox.umd';

const source = "public.poi_osm_light";

const layers = [
  {
    id: "poi-open-background",
    type: "circle",
    "source-layer": source,
    filter: [
      "all",
      [
        "in",
        "status",
        "open",
        "open_adapted"
      ]
    ],
    paint: {
      'circle-color': 'green',
      'circle-radius': [
        'interpolate',
        ['linear'],
        ['zoom'],
        14, 6,
        19, 17
      ]
    }
  },
  {
    id: "poi-unknow-background",
    type: "circle",
    "source-layer": source,
    filter: [
      "all",
      [
        "in",
        "status",
        "unknown",
        "partial"
      ]
    ],
    paint: {
      'circle-color': 'gray',
      'circle-radius': [
        'interpolate',
        ['linear'],
        ['zoom'],
        14, 6,
        19, 17
      ]
    }
  },
  {
    id: "poi-closed-background",
    type: "circle",
    "source-layer": source,
    filter: [
      "all",
      [
        "in",
        "status",
        "closed"
      ]
    ],
    paint: {
      'circle-color': '#96281b',
      'circle-radius': [
        'interpolate',
        ['linear'],
        ['zoom'],
        14, 6,
        19, 17
      ]
    }
  },
  {
    id: "poi-white-bg",
    type: "circle",
    "source-layer": source,
    paint: {
      'circle-color': 'white',
      'circle-radius': [
        'interpolate',
        ['linear'],
        ['zoom'],
        14, 3,
        19, 14
      ]
    }
  },
  {
    id: "poi-icon",
    type: "symbol",
    "source-layer": source,
    minzoom: 15,
    "layout": {
      "icon-image": [
        "coalesce",
        ['image', ['concat', ['get', 'cat'], '_11']],
        ['image', ['concat', ['get', 'normalized_cat'], '_11']],
        ['image', 'other_11']
      ],
      "icon-size": [
        'interpolate',
        ['linear'],
        ['zoom'],
        14, 0.3,
        19, 0.9
      ],
      "text-anchor": "top",
      "text-field": ["get", "name"],
      "text-font": [
        "Noto Sans Regular"
      ],
      "text-max-width": 9,
      "text-offset": [
        'interpolate',
        ['linear'],
        ['zoom'],
        14, ['literal', [0, 0]],
        19, ['literal', [0, 1.5]]
      ],
      "text-padding": 2,
      "text-size": 12
    },
    paint: {
      "text-color": "#666",
      "text-halo-blur": 0.5,
      "text-halo-color": "#ffffff",
      "text-halo-width": 1
    }
  }
];

export default {
  components: {
    MglAttributionControl,
    MglMap,
    MglNavigationControl,
    MglVectorLayer,
    MglGeolocateControl
  },

  props: {
    featuresAndLocation: {
      type: String,
      required: false,
      default: ''
    },

    mapStyle: {
      type: Object,
      required: true
    },

    mapCenter: {
      type: Object,
      required: true
    },

    mapZoom: {
      type: Number,
      required: true
    },

    filter: {
      type: String,
      required: true
    }
  },

  mounted() {
    this.$on('updateMapBounds', (bbox) => {
      this.map.fitBounds(bbox, { duration: 0 });
    });
  },

  computed: {
    poiSource() {
      return {
        minzoom: 13,
        maxzoom: 13,
        tiles: [config.tilesUrl]
      };
    },

    layers() {
      return layers.map((layer) => {
        const newLayer = { ...layer, filter: layer.filter ? [...layer.filter] : [ "all" ] };

        if (this.filter !== '') {
          newLayer.filter.push([
            "==",
            "normalized_cat",
            this.filter
          ]);
        }

        return newLayer;
      });
    }
  },

  methods: {
    load({ map }) {
      this.map = map;
    },

    updateMapCenter(mapCenter) {
      this.$emit('update:mapCenter', mapCenter);
    },

    updateMapZoom(mapZoom) {
      this.$emit('update:mapZoom', mapZoom);
    },

    mouseenter(e) {
      e.map.getCanvas().style.cursor = 'pointer';
    },

    mouseleave(e) {
      e.map.getCanvas().style.cursor = '';
    },

    clickPoi(e) {
      const id = e.mapboxEvent.features[0].properties.fid;
      if (this.$route.name === 'place' && this.$route.params.id === id) {
        return;
      }
      this.$router.push({
        name: 'place',
        params: {
          id,
          featuresAndLocation: this.featuresAndLocation
        }
      });
    }
  }
}
</script>

<style>
.mgl-map-wrapper {
  width: 100vw;
  height: 100vh;
  position: relative;
}

.mgl-map-wrapper .mapboxgl-map {
  height: 100%;
  left: 0;
  position: absolute;
  top: 0;
  width: 100%;
}
</style>
