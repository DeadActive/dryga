<template>
  <div id="app">
    <div class="left">
      <form class="left-form">
        <div class="input-box-wrap">
          <input
            type="number"
            step="1"
            name="lx"
            id="lx"
            class="input-box i-gsd"
            v-model="lx"
            required
          />
          <label for="lx" class="input-box-placeholder">Ширина снимка</label>
        </div>
        <div class="input-box-wrap">
          <input
            type="number"
            step="1"
            name="ly"
            id="ly"
            class="input-box i-gsd"
            v-model="ly"
            required
          />
          <label for="ly" class="input-box-placeholder">Высота снимка</label>
        </div>
        <div class="input-box-wrap">
          <input
            type="number"
            step="0.00001"
            name="gsd"
            id="gsd"
            class="input-box i-gsd"
            v-model="gsd"
            required
          />
          <label for="gsd" class="input-box-placeholder">GSD</label>
        </div>
        <div class="input-box-wrap">
          <input
            type="number"
            name="long"
            id="long"
            class="input-box i-longover"
            step="0.1"
            v-model="px"
            required
          />
          <label for="long" class="input-box-placeholder"
            >Продольное перекрытие</label
          >
        </div>
        <div class="input-box-wrap">
          <input
            type="number"
            name="traverse"
            id="traverse"
            class="input-box i-traverseover"
            step="0.1"
            v-model="py"
            required
          />
          <label for="traverse" class="input-box-placeholder"
            >Поперечное перекрытие</label
          >
        </div>
        <div class="input-box-wrap">
          <input
            type="number"
            name="vel"
            id="vel"
            class="input-box i-longover"
            step="0.1"
            v-model="vel"
            required
          />
          <label for="vel" class="input-box-placeholder">Скорость</label>
        </div>
      </form>
    </div>
    <div class="right">
      <div class="map-wrapper">
        <l-map
          :zoom="zoom"
          :center="center"
          class="map"
          ref="map"
          @click="clickLocation($event)"
          @mousedown="mouseDown($event)"
          @mouseup="mouseUp($event)"
        >
          <l-control class="toolbar" position="topleft">
            <div class="rect-button" @click="drawRect()">
              🔲
            </div>
          </l-control>
          <l-control class="params">
            <p>
              GSD: {{ gsd }} м/пкс<br />
              Высота полета: {{ flightHeight }} м<br />
              Базис: {{ flightBasic.toFixed(2) }} м<br />
              Поперечный базис: {{ flightSpacing.toFixed(2) }} м<br />
            </p>

            <p v-if="rectDrawn">
              Ширина: {{ rectSize.width.toFixed(2) }} м<br />
              Высота: {{ rectSize.height.toFixed(2) }} м<br />
              Число снимков: {{ shotsCount }}<br />
              Число маршрутов: {{ routesCount }}<br />
              Среднее время полета: {{ estimateTime }}<br />
              Интервал фотографирования: {{ shotsInterval.toFixed(1) }} c
            </p>
          </l-control>
          <l-tile-layer :url="url"></l-tile-layer>
          <l-rectangle
            v-if="rectEnabled"
            ref="rect"
            :bounds="bounds"
          ></l-rectangle>
          <l-polyline
            v-if="rectEnabled"
            ref="route"
            :latLngs="routeLatlngs"
            color="#ff0000"
            @click="getGeojson($event)"
          ></l-polyline>
        </l-map>
      </div>
    </div>
  </div>
</template>

<script>
import 'normalize.css'
import {
  LMap,
  LTileLayer,
  LControl,
  LRectangle,
  LPolyline,
  LMarker,
} from 'vue2-leaflet'
import { isNull } from 'util'

export default {
  name: 'app',
  data() {
    return {
      zoom: 13,
      center: L.latLng(55.14454538318123, 37.456169128417976),
      url: 'http://{s}.tile.osm.org/{z}/{x}/{y}.png',
      gsd: 0.02,
      bounds: [],
      routeLatlngs: [],
      isDraw: false,
      rectEnabled: false,
      rectDrawn: false,
      dragging: false,
      px: 60,
      py: 20,
      focus: 0,
      vel: 20,
      lx: 4000,
      ly: 3000,
    }
  },
  components: { LMap, LTileLayer, LControl, LRectangle, LPolyline, LMarker },
  methods: {
    clickLocation: function(event) {
      //console.log(event.latlng)
    },
    dragEnabled: function() {
      return this.$refs.map.mapObject.dragging._enabled
    },
    drawRect: function() {
      this.dragEnabled()
        ? this.$refs.map.mapObject.dragging.disable()
        : this.$refs.map.mapObject.dragging.enable()

      this.isDraw = !this.isDraw

      // isNull(this.rect)
      //   ? (this.rect = LRectangle(this.$refs.map.mapObject.getBounds(), {
      //       color: '#ff7800',
      //       weight: 1,
      //     }))
      //   : null
    },
    mouseDown: function(e) {
      if (this.isDraw) {
        let latlng = e.latlng
        this.bounds = []
        this.routeLatlngs = []
        this.bounds.push([latlng.lat, latlng.lng])
        this.rectEnabled = true
        this.rectDrawn = false
        this.dragging = true
      }
    },
    mouseUp: function(e) {
      if (this.isDraw) {
        let latlng = e.latlng
        this.bounds.push([latlng.lat, latlng.lng])
        this.isDraw = false
        this.$refs.map.mapObject.dragging.enable()
        this.rectDrawn = true
        this.dragging = true
        this.createRoute()
      }
    },
    pointToLatlng: function(point) {
      console.log(this.$refs.map.mapObject.layerPointToLatLng(point))
      return this.$refs.map.mapObject.layerPointToLatLng(point)
    },
    createRoute: function() {
      this.routeLatlngs = []
      let size = this.rectSize
      let tl = this.bounds[0]
      let br = this.bounds[1]
      let R = 6378137
      let dLat = this.flightSpacing / R

      for (let i = 0; i < this.routesCount; i++) {
        let dLon =
          this.flightBasic /
          (R * Math.cos((Math.PI * (tl[0] - (dLat * i * 180) / Math.PI)) / 180))
        for (let j = 0; j < this.shotsCount; j++) {
          let n = i % 2 == 0 ? j : this.shotsCount - j - 1

          this.routeLatlngs.push([
            tl[0] - (dLat * (i - 0.5) * 180) / Math.PI,
            tl[1] + (dLon * (n - 1.5) * 180) / Math.PI,
          ])
        }
      }
    },
    getGeojson: function(e) {
      let geojson = JSON.stringify(e.sourceTarget.toGeoJSON())

      const el = document.createElement('textarea') // Create a <textarea> element
      el.value = geojson // Set its value to the string that you want copied
      el.setAttribute('readonly', '') // Make it readonly to be tamper-proof
      el.style.position = 'absolute'
      el.style.left = '-9999px' // Move outside the screen to make it invisible
      document.body.appendChild(el) // Append the <textarea> element to the HTML document
      const selected =
        document.getSelection().rangeCount > 0 // Check if there is any content selected previously
          ? document.getSelection().getRangeAt(0) // Store selection if found
          : false // Mark as false to know no selection existed before
      el.select() // Select the <textarea> content
      document.execCommand('copy') // Copy - only works as a result of a user action (e.g. click events)
      document.body.removeChild(el) // Remove the <textarea> element
      if (selected) {
        // If a selection existed before copying
        document.getSelection().removeAllRanges() // Unselect everything on the HTML document
        document.getSelection().addRange(selected) // Restore the original selection
      }

      alert('GeoJSON скопирован в буфер обмена')
    },
  },
  computed: {
    rectSize: function() {
      let tl = this.bounds[0]
      let br = this.bounds[1]

      let wh = {
        width: L.latLng(tl).distanceTo(L.latLng(tl[0], br[1])),
        height: L.latLng(tl).distanceTo(L.latLng(br[0], tl[1])),
      }
      return wh
    },
    getDx: function() {
      let tl = this.bounds[0]
      let br = this.bounds[1]
      return L.latLng(tl).distanceTo(L.latLng(tl[0], br[1]))
    },
    getDy: function() {
      let tl = this.bounds[0]
      let br = this.bounds[1]
      return L.latLng(tl).distanceTo(L.latLng(br[0], tl[1]))
    },
    flightHeight: function() {
      return this.gsd * this.lx
    },
    flightBasic: function() {
      return ((this.lx * (100 - this.px)) / 100) * this.gsd
    },
    flightSpacing: function() {
      return ((this.ly * (100 - this.py)) / 100) * this.gsd
    },
    shotsCount: function() {
      return Math.ceil(this.getDx / this.flightBasic + 3)
    },
    routesCount: function() {
      return Math.ceil(this.getDy / this.flightSpacing + 1)
    },
    estimateTime: function() {
      let time = new Date(null)
      let seconds = Math.ceil(
        (this.routesCount * (this.getDx + 3 * this.flightBasic)) / this.vel
      )
      time.setSeconds(seconds)

      return time.toISOString().substr(11, 8)
    },
    shotsInterval: function() {
      return this.flightBasic / this.vel
    },
  },
  mounted() {},
}
</script>

<style scoped lang="scss">
@import './assets/scss/colors.scss';

body {
  background: $color-main;
}

.map {
  // width: 100%;
  // height: 100%;
}

#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-direction: row;
  width: 100%;
  background: $color-main;
}

.toolbar {
  background: #fff;
  padding: 10px;
  border-radius: 10px;
  margin-top: 200px;
}

.rect-button {
  cursor: pointer;
}

.params {
  margin: 0;
  margin-right: 20px;
  display: block;
  background: #fff;
  font-size: 16px;
  padding: 5px 10px;
  border-radius: 10px;
}

.left {
  display: flex;
  margin: 0;
  padding: 0;
  justify-content: center;
  align-items: flex-start;
  flex-direction: column;
}

.left-form {
  margin: 0;
  margin-left: 30px;
  margin-right: 30px;
}

form {
  display: flex;
  justify-content: center;
  align-content: center;
  flex-direction: column;
}

.map-wrapper {
  width: 100%;
  height: 100vh;
}

.right {
  display: flex;
  justify-content: center;
  align-items: center;
  background: red;
  flex-grow: 1;
}

.input-box-wrap {
  position: relative;
  margin-bottom: 20px;
  margin-right: 22px;
  flex-grow: 1;
  transition: all 0.5s;
  color: #fff;
}

.input-box {
  background: $color-alt;
  border: none;
  border-radius: 6px;
  min-width: 200px;
  height: 4rem;
  color: #fff;
  padding-left: 1rem;
  line-height: 0;

  &:focus {
    outline: 0;
    box-shadow: 0;
    padding-top: 1rem;
  }
  &:focus + label {
    top: 1rem;
    font-size: 0.8rem;
    color: $color-acc;
  }

  &::placeholder {
    color: transparent;
  }

  &:valid {
    padding-top: 1rem;
  }

  &:valid + label {
    top: 1rem;
    font-size: 0.8rem;
    color: $color-acc;
  }
}

.input-box-placeholder {
  color: $text-light;
  font-size: 1rem;
  position: absolute;
  top: 2rem;
  line-height: 0;
  bottom: 0;
  left: 1rem;
  transition: 0.3s;
}
</style>
