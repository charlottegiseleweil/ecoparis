<template>
  <div class="map-container">
    <section ref="map" class="map"></section>

    <svg ref="circle_svg" class="EV-circle-svg"></svg> 
    <svg ref="svg" class="EV-svg"></svg>

    <section class="year" v-if="hasFutureAvailable && !isFuture">
      <p>2012</p>
      <a href="#" @click.prevent="() => $emit('toggleFuture')"
        v-bind:style="{'color': futureColor}">
        <i class="fas fa-fast-forward"></i>
      </a>
    </section>
    <section class="year" v-else-if="hasFutureAvailable">
      <p>2030</p>
      <a href="#" @click.prevent="() => $emit('toggleFuture')"
        v-bind:style="{'color': presentColor}">
        <i class="fas fa-fast-backward"></i>
      </a>
    </section>

    <section class="legend">
      <div ref="legend" class="legend-inner"></div>

      <section v-if="isEspacesVerts">
        <h3>{{ $t('legend.natural-spaces') }}</h3>

        <figure>
          <div class="item" :style="{backgroundColor: forestColor}"></div>
          <figcaption v-html="$t('legend.wood')"></figcaption>
        </figure>
        <figure>
          <div class="item" :style="{backgroundColor: waterColor}"></div>
          <figcaption v-html="$t('legend.water')"></figcaption>
        </figure>
        <figure>
          <div class="item" :style="{backgroundColor: greenColor}"></div>
          <figcaption v-html="$t('legend.leisure')"></figcaption>
        </figure>
      </section>

      <section v-else-if="currentUnit">
        <p>({{ currentUnit }})</p>
      </section>
    </section>
  </div>
</template>

<script>
import Config from '../config.json'
import displayMap from '../map'
import helpers_f from '../helpers.js'

export default {
  props: ['layers', 'currentLayerPath', 'currentLocation', 'currentZoom', 'isFuture'],
  data: () => ({
    loadLayer: () => {},
    setLayer: () => {},
    setLocation: () => {},
    setEVLayer: () => {},
    zoomMinus: () => {},
    zoomPlus: () => {},
  }),

  computed: {
    isEspacesVerts() {
      return this.currentLayerPath == Config.EV_path
    },
    
    currentLayerId() {
      return Config.layers[this.currentLayerPath].id
    },

    currentUnit() {
      if (this.currentLayerPath) {
        return this.$t(`layers.${this.currentLayerId}.unit`)
      }
    },

    forestColor() {
      return Config.layers[Config.EV_path].colors[0]
    },

    waterColor() {
      return Config.layers[Config.EV_path].colors[1]
    },

    greenColor() {
      return Config.layers[Config.EV_path].colors[2]
    },

    futureColor() {
      return Config.color_future
    },

    presentColor() {
      return Config.color_present
    },

    currentYear() {
      return new Date().getFullYear()
    },

    hasFutureAvailable() {
      return !!Config.layers[this.currentLayerPath].future
    }
  },

  /**
   * Triggers when the component has been mounted.
   * This is used to create the Leaflet map and bind the event handlers.
   */
  mounted () {
    var urlInterComm = 'data/intercommunalites.geojson'
    var urlVoronoi = 'data/voronois.json'
    var urlInterCommContainment = 'data/intercomm_cont.json'
    var urlVoronoiContainment = 'data/voronoi_cont.json'

    queue()
      .defer(d3.json, urlInterComm)
      .defer(d3.json, urlVoronoi)
      .await((e, d, v) => {
        helpers_f.loadContainmentFile(urlVoronoiContainment).then(voronoiContainment => {
          helpers_f.loadContainmentFile(urlInterCommContainment).then(interCommContainment => {
            // FIXME(liautaud): Please, clean up this mess.
            [this.loadLayer, this.setLayer, this.setLocation, this.setEVLayer, this.zoomMinus, this.zoomPlus] =
              displayMap(
                this.$refs.map,
                this.$refs.svg,
                this.$refs.circle_svg,
                this.$refs.legend,
                e, d, v, interCommContainment, voronoiContainment,
                (x, y) => this.$emit('newHistogram', x, y),
                n => this.$emit('newSchools', n),
                n => this.$emit('newName',n)
              )

            Object.keys(Config.layers).forEach(layerPath => {
              this.layers[layerPath].path=layerPath;

              // Preload all the layers.
              this.loadLayer(
                layerPath,
                false,
                () => {
                  this.layers[layerPath].loaded = true
                  
                  if (layerPath == Config.EV_path){
                    //this.setEVLayer(layerPath)
                  }          
                }
              )

              // Also preload the future layers when applicable.
              if (Config.layers[layerPath].future) {
                console.log('Preloading future layer:', Config.layers[layerPath].future)
                this.loadLayer(layerPath, true, () => {})
              }
            })

            this.setLayer(this.currentLayerPath)
          })
        })
      })
  },

  watch: {
    /**
     * Watches changes to the currentLayerPath prop, and updates the map.
     */
    currentLayerPath() {
      this.setLayer(this.currentLayerPath, this.hasFutureAvailable && this.isFuture)
    },

    /**
     * Watches changes to the isFuture prop, and updates the map.
     */
    isFuture() {
      this.setLayer(this.currentLayerPath, this.hasFutureAvailable && this.isFuture)
    },

    /**
     * Watches changes to the currentLocation prop, and updates the marker.
     */
    currentLocation([lat, lng]) {
      this.setLocation(lat, lng)
    },

    /**
     * Watches changes to the currentZoom prop, and updates the marker.
     */
    currentZoom(after, before) {
      // FIXME(liautaud): This only takes +/-1 changes into account.
      // It's a dirty hack anyway, we should hold the currentZoom instead.
      if (after < before) {
        this.zoomMinus()
      } else {
        this.zoomPlus()
      }
    }
  }
}
</script>

<style lang="scss" type="text/scss">
.map-container {
  flex-grow: 1;
  position: relative;
}

.map {
  width: 100%;
  height: 100%;
}

.leaflet-overlay-pane {
  mix-blend-mode: multiply;

  image {
    image-rendering: optimizespeed;
  }
}

.leaflet-top, .leaflet-bottom {
  z-index: 300 !important;
}

.EV-svg {
  border-radius: 50%;
  width: 120px;
  height: 120px;
  border: 3px solid;
  border-color:#000;
  box-shadow: 0 0 3px rgba(#000, .3);
  position: absolute;
  background: #fff;
  display: flex;
  pointer-events: none;
  top: -120px;
  margin-left: -20px;
  margin-bottom: -20px;

  image {
    image-rendering: optimizespeed;
  }
}

.EV-circle-svg {
  border-radius: 50%;
  width: 80px;
  height: 80px;
  border: 2px dashed;
  border-color:rgba(#000,0.7);
  position: absolute;
  background: #fff0;
  display: flex;
  pointer-events: none;
  top: -120px;
}

.year {
  position: absolute;
  top: 35px;
  right: 25px;

  box-shadow: 0 0 2px rgba(0, 0, 0, .3);
  background: #fff;

  display: flex;

  p {
    margin: 5px 20px;
    color: #666;
    font-size: 25pt;
    font-family: 'IBM Plex Sans', sans-serif;
  }

  a {
    display: flex;
    align-items: center;
    border-left: 1px solid #ddd;
    font-size: 18pt;
    outline: none;
    text-decoration: none;
    padding: 15px;
    color: #444;
  }
}

.legend {
  position: absolute;
  bottom: 40px;
  right: 30px;
  text-align: center;
  padding: 5px 10px;

  box-shadow: 0 0 2px rgba(0, 0, 0, .3);
  background: #fff;

  h3 {
    text-transform: uppercase;
    margin: 5px;
  }

  p, figcaption {
    font-size: .6em;
    font-style: italic;
    margin: 0 auto;
  }

  figure {
    margin: 15px 10px;
  }

  .item {
    width: 70px;
    height: 28px;
    border: 1px solid #000;
    margin: 5px auto;
  }
}

.legend-inner {
  height: 200px;
  width: 70px;
  margin: 0 auto 10px;
  position: relative;
}

.legend-inner canvas {
  box-sizing: border-box;
  border: 1px solid #000;
  border-right: 0;
}
</style>
