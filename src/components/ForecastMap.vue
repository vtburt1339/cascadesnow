<template>
  <div ref="mapEl"
       class="map"></div>
</template>

<script>
import { onBeforeUnmount, onMounted, ref, watch, nextTick } from 'vue'

export default {
  name: 'ForecastMap',
  props: {
    geometry: {
      type: Object,
      default: null
    },
    overlay: {
      type: Object,
      default: null
    }
  },
  setup(props) {
    const mapEl = ref(null)
    let map = null
    let geoLayer = null
    let overlayLayer = null
    let resizeHandler = null

    const invalidateMap = () => {
      if (!map) return
      requestAnimationFrame(() => map.invalidateSize())
    }

    const initMap = () => {
      const L = window.L
      if (!L || !mapEl.value) return

      map = L.map(mapEl.value, { zoomControl: false }).setView([44.05, -121.8], 8)
      L.tileLayer('https://toposm.ahlzen.com/tiles/hikemap_composite_20240618/{z}/{x}/{y}.png', {
        maxZoom: 15,
        attribution: '&copy; OpenStreetMap contributors'
      }).addTo(map)

      invalidateMap()
      setTimeout(invalidateMap, 120)
    }

    const updateGeometry = (geometry) => {
      const L = window.L
      if (!L || !map) return

      if (geoLayer) {
        geoLayer.remove()
        geoLayer = null
      }

      if (!geometry) return

      geoLayer = L.geoJSON(
        { type: 'Feature', geometry },
        {
          style: {
            color: '#2a7f74',
            weight: 2,
            fillColor: '#2a7f74',
            fillOpacity: 0.12
          }
        }
      ).addTo(map)

      map.fitBounds(geoLayer.getBounds(), { padding: [20, 20] })
      invalidateMap()
    }

    const updateOverlay = (overlay) => {
      const L = window.L
      if (!L || !map) return

      if (overlayLayer) {
        overlayLayer.remove()
        overlayLayer = null
      }

      if (!overlay) return

      overlayLayer = L.geoJSON(overlay, {
        style: {
          color: 'purple',
          weight: 2,
          fillColor: 'purple',
          fillOpacity: 0.35
        }
      }).addTo(map)
      invalidateMap()
    }

    onMounted(() => {
      initMap()
      resizeHandler = () => invalidateMap()
      window.addEventListener('resize', resizeHandler)
    })

    onBeforeUnmount(() => {
      if (resizeHandler) {
        window.removeEventListener('resize', resizeHandler)
      }
      if (map) {
        map.remove()
      }
    })

    watch(
      () => props.geometry,
      async (nextGeometry) => {
        await nextTick()
        updateGeometry(nextGeometry)
      },
      { immediate: true }
    )

    watch(
      () => props.overlay,
      async (nextOverlay) => {
        await nextTick()
        updateOverlay(nextOverlay)
      },
      { immediate: true }
    )

    return {
      mapEl
    }
  }
}
</script>

<style scoped>
.map {
  width: 100%;
  height: 100%;
  border-radius: 18px;
  overflow: hidden;
}

@media (max-width: 980px) {
  .map {
    height: 100%;
  }
}
</style>
