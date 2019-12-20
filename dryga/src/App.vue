<template>
  <div id="app">
    <div class="left">
      <form class="left-form">
        <div class="input-box-wrap">
          <input
            type="number"
            step="0.001"
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
            name="focal"
            id="focal"
            class="input-box i-focal"
            required
          />
          <label for="focal" class="input-box-placeholder"
            >Фокусное расстояние</label
          >
        </div>
        <div class="input-box-wrap">
          <input
            type="number"
            name="long"
            id="long"
            class="input-box i-longover"
            step="0.1"
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
            required
          />
          <label for="traverse" class="input-box-placeholder"
            >Поперечное перекрытие</label
          >
        </div>
        <input type="submit" value="Посчитать" />
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
            GSD: {{ gsd }} пкс/м
            <br />
            <p v-if="rectDrawn">
              Ширина: {{ rectSize.width.toFixed(2) }} м<br />
              Высота: {{ rectSize.height.toFixed(2) }} м
            </p>
          </l-control>
          <l-tile-layer :url="url"></l-tile-layer>
          <l-rectangle
            v-if="rectEnabled"
            ref="rect"
            :bounds="bounds"
          ></l-rectangle>
        </l-map>
      </div>
    </div>
  </div>
</template>

<script>
import 'normalize.css'
import { LMap, LTileLayer, LControl, LRectangle } from 'vue2-leaflet'
import { isNull } from 'util'

export default {
  name: 'app',
  data() {
    return {
      zoom: 13,
      center: L.latLng(55.14454538318123, 37.456169128417976),
      url: 'http://{s}.tile.osm.org/{z}/{x}/{y}.png',
      gsd: 0,
      bounds: [],
      isDraw: false,
      rectEnabled: false,
      rectDrawn: false,
      dragging: false,
    }
  },
  components: { LMap, LTileLayer, LControl, LRectangle },
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
      }
    },
    pointToLatlng: function(point) {
      console.log(this.$refs.map.mapObject.layerPointToLatLng(point))
      return this.$refs.map.mapObject.layerPointToLatLng(point)
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
