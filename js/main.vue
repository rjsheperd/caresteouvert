<template>
  <div
    v-resize="computeIsMobile"
    :class="{
       xs: $vuetify.breakpoint.xsOnly,
      'place-opened': $route.name === 'place' && !isMobile,
      'sidebar-opened': sidebar && !isMobile
     }"
    >
    <div>
      <v-navigation-drawer
        v-model="sidebar"
        :temporary="isMobile"
        :stateless="!isMobile"
        :hide-overlay="!isMobile || !sidebar"
        width="300"
        fixed
      >
        <osm-sidebar>
          <osm-filter-features
            v-model="filter"
            :categories="categories"
          />
        </osm-sidebar>
      </v-navigation-drawer>
      <v-content>
        <top-toolbar
          @toggleSidebar="sidebar = !sidebar"
          @onGeocode="updateMapBounds"
        />
        <osm-map
          v-if="loadMap"
          ref="map"
          :map-style="mapStyle"
          :map-center.sync="mapCenter"
          :map-zoom.sync="mapZoom"
          :filter="filter"
          :featuresAndLocation="featuresAndLocation"
        />
        <rgpd-banner />
      </v-content>
    </div>
    <router-view />
  </div>
</template>

<script>
import * as config from '../config.json';
import * as categories from '../categories.json';
import { encode, decode, encodePosition, decodePosition } from './url';
import OsmSidebar from './sidebar';
import OsmFilterFeatures from './filter_features';
import OsmMap from './map';
import TopToolbar from './top_toolbar';
import RgpdBanner from './rgpd_banner';

export default {
  components: {
    OsmFilterFeatures,
    OsmSidebar,
    OsmMap,
    TopToolbar,
    RgpdBanner
  },

  props: {
    featuresAndLocation: {
      type: String,
      required: false,
      default: ''
    }
  },

  data() {
    return {
      loadMap: false,
      isMobile: false,
      sidebar: false,
      mapStyle: null,
      mapCenter: null,
      mapZoom: null,
      mapStyle: `${config.mapStyle}${config.apiKey}`,
      filter: '',
      categories: Object.keys(categories.categories).concat([ "other" ])
    };
  },

  mounted() {
    this.computeIsMobile();

    this.sidebar = !this.isMobile;

    const { filter, location } = decode(this.featuresAndLocation);
    this.filter = this.categories.includes(filter) ? filter : '';

    Promise.all([
      this.loadInitialLocation(location),
      this.loadAndOverrideMapStyle()
    ]).then(() => { this.loadMap = true; });
  },

  watch: {
    mapCenter() {
      this.updateRoute();
      this.saveCurrentView();
    },

    mapZoom() {
      this.updateRoute();
      this.saveCurrentView();
    },

    filter() {
      this.updateRoute();
    }
  },

  methods: {
    loadAndOverrideMapStyle() {
      function createWebUrl(url) {
        const a = document.createElement("a")
        a.href = url
        // Fix populating Location properties in IE. Otherwise, protocol will be blank.
        a.href = a.href
        return a.href
      }
      return fetch(`${config.mapStyle}${config.apiKey}`)
        .then(res => res.json())
        .then((data) => {
          data.sprite = createWebUrl('/sprite/caresteouvert');
          this.mapStyle = data;
        });
    },

    loadInitialLocation(location) {
      let promise;
      const savedMapView = this.savedMapView();
      if (location) {
        const { lat, lng, zoom } = decodePosition(location, config);
        this.mapCenter = { lat, lng };
        this.mapZoom = zoom;
        promise = Promise.resolve();
      } else if (savedMapView) {
        const { center, zoom } = JSON.parse(savedMapView);
        this.mapCenter = center;
        this.mapZoom = zoom;
        promise = Promise.resolve();
      } else {
        promise = this.centerMapViaGeoIP();
      }
      return promise;
    },

    centerMapViaGeoIP() {
      return fetch(config.geoIpUrl)
        .then(res => res.json())
        .then(json => {
          this.mapCenter = { lat: json.ll[0], lng: json.ll[1] };
          this.mapZoom = 13;
        }).catch(() => {
          this.mapCenter = { lat: config.mapCenter[1], lng: config.mapCenter[0] };
          this.mapZoom = config.mapZoom;
        });
    },

    updateRoute() {
      const currentFeaturesAndLocation = this.$route.params.featuresAndLocation;
      const newFeaturesAndLocation = encode(
        this.filter,
        encodePosition(this.mapCenter.lat, this.mapCenter.lng, this.mapZoom)
      );
      if (currentFeaturesAndLocation === newFeaturesAndLocation) {
        return;
      }
      this.$router.replace({
        name: this.$route.name,
        params: {
          ...this.$route.params,
          featuresAndLocation: newFeaturesAndLocation
        }
      });
    },

    updateMapBounds(bbox) {
      this.$refs.map.$emit('updateMapBounds', bbox);
    },

    computeIsMobile() {
      this.isMobile = window.innerWidth < 800;
    },

    savedMapView() {
      return localStorage.getItem('mapView');
    },

    saveCurrentView() {
      localStorage.setItem('mapView', JSON.stringify({ center: this.mapCenter, zoom: this.mapZoom }));
    }
  }
}
</script>

<style>
.xs .mapboxgl-ctrl-top-right {
  top: 50px;
}
.place-opened .mapboxgl-ctrl-top-right, .place-opened .mapboxgl-ctrl-bottom-right {
  transform: translateX(-400px);
}
.sidebar-opened .search {
  transform: translateX(300px);
}
</style>
