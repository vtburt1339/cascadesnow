<template>
  <div class="app"
       style="position:fixed; inset:0; overflow:hidden;">
    <header class="header">
      <div class="title-wrap">
        <h1>Cascade Forecast Snow Line</h1>
      </div>
      <div class="controls">
        <div class="zone-buttons">
          <button type="button"
                  :class="{ active: zoneId === 'ORZ128' }"
                  @click="selectZone('ORZ128')">
            Cascades of Lane County
          </button>
          <button type="button"
                  :class="{ active: zoneId === 'ORZ127' }"
                  @click="selectZone('ORZ127')">
            Cascades of Marion and Linn Counties
          </button>
        </div>
      </div>
    </header>

    <main class="grid">
      <section class="panel map-panel">
        <div class="meta">
          <div>
            <strong>NOAA Forecast Zone: </strong>
            <a :href="`https://forecast.weather.gov/MapClick.php?zoneid=${zoneId}`"
               target="_blank"
               rel="noopener noreferrer">
              {{ (zoneId === 'ORZ128' ? 'Cascades of Lane County' : zoneId === 'ORZ127'
                ? 'Cascades of Marion and Linn Counties' : 'Unknown') + ' (' + zoneId + ')' }}
            </a>
          </div>
        </div>
        <ForecastMap :geometry="geometry"
                     :overlay="overlayGeojson" />
      </section>

      <section class="panel forecast-panel">
        <div v-if="loading"
             class="loading-bar"></div>
        <div v-if="error"
             class="error">{{ error }}</div>
        <div v-if="updated"><strong>Updated:</strong> {{ formattedUpdated }}</div>
        <div v-else
             class="status">Waiting for forecast data.</div>
        <template v-if="!loading && !error && periods.length">
          <ForecastGraph :graph-periods="graphPeriods"
                         :slider-count="displayPeriods.length"
                         :slider-index="selectedIndex"
                         :slider-label="displayPeriods[selectedIndex]?.name || ''"
                         :snow-level="snowLevelForPeriod(displayPeriods[selectedIndex])"
                         @update:sliderIndex="selectPeriod" />


          <div class="cards">
            <article v-for="(period, index) in displayPeriods"
                     :key="period.number"
                     class="card"
                     :class="{
                      active: selectedIndex === index,
                      'is-clickable': snowLevelForPeriod(period)
                    }"
                     :style="{ animationDelay: `${index * 80}ms` }"
                     @click.stop="selectPeriod(index)">
              <h3>{{ period.name }}</h3>
              <p>{{ period.detailedForecast }}</p>
              <div class="details">
                <div v-if="snowLevelForPeriod(period)">
                  Snow level: {{ snowLevelForPeriod(period) }} ft
                </div>
                <div v-if="extractChance(period.detailedForecast)">
                  Precip chance: {{ extractChance(period.detailedForecast) }}
                </div>
              </div>
            </article>
          </div>
        </template>

        <div v-if="!loading && !error && !periods.length"
             class="status">
          No forecast periods returned for this zone.
        </div>
      </section>
    </main>
  </div>
</template>

<script>
import ForecastMap from './components/ForecastMap.vue'
import ForecastGraph from './components/ForecastGraph.vue'

export default {
  name: 'App',
  components: {
    ForecastMap,
    ForecastGraph
  },
  data() {
    return {
      zoneId: '',
      loading: false,
      error: '',
      periods: [],
      updated: '',
      geometry: null,
      overlayGeojson: null,
      selectedSnowLevel: '',
      selectedIndex: 0,
      overlayRequestId: 0
    }
  },
  computed: {
    formattedUpdated() {
      if (!this.updated) return ''
      return new Date(this.updated).toLocaleString()
    },
    displayPeriods() {
      return this.periods.slice(0, 8)
    },
    graphPeriods() {
      return this.displayPeriods.map((period) => ({
        name: period?.name || '',
        snowLevel: Number(this.snowLevelForPeriod(period)) || null,
        precipChance: this.extractChanceValue(period?.detailedForecast || '')
      }))
    }
  },
  mounted() {
    const savedZone = localStorage.getItem('cascade-zone-id')
    this.zoneId = savedZone || 'ORZ128'
    const savedIndex = this.getSavedIndex(this.zoneId)
    this.fetchForecast(savedIndex)
  },
  methods: {
    getSavedIndex(zoneId) {
      const raw = localStorage.getItem(`cascade-forecast-day-${zoneId}`)
      const parsed = Number(raw)
      return Number.isFinite(parsed) ? parsed : 0
    },
    selectZone(zoneId) {
      if (this.zoneId === zoneId) return
      const carriedIndex = this.selectedIndex
      this.zoneId = zoneId
      localStorage.setItem('cascade-zone-id', zoneId)
      this.fetchForecast(carriedIndex)
    },
    extractSnowLevel(text) {
      const match = text.match(/Snow level(?:\s+above|\s+around)?\s*([0-9,]+)\s+feet/i)
      return match ? match[1].replace(/,/g, '') : ''
    },
    snowLevelForPeriod(period) {
      if (!period || !period.detailedForecast) return ''
      return this.extractSnowLevel(period.detailedForecast)
    },
    clearOverlay() {
      if (!this.overlayGeojson && !this.selectedSnowLevel) return
      this.overlayGeojson = null
      this.selectedSnowLevel = ''
      this.overlayRequestId += 1
    },
    async selectPeriod(index) {
      const nextIndex = Number.isFinite(index) ? index : Number(index)
      this.selectedIndex = Number.isNaN(nextIndex) ? 0 : nextIndex
      localStorage.setItem(`cascade-forecast-day-${this.zoneId}`, String(this.selectedIndex))
      const period = this.displayPeriods[this.selectedIndex]
      if (!period) {
        this.clearOverlay()
        return
      }

      const level = this.snowLevelForPeriod(period)
      if (!level) {
        this.clearOverlay()
        return
      }

      if (this.selectedSnowLevel === level && this.overlayGeojson) return

      const assetUrl = `${process.env.BASE_URL}thresholds/${level}.geojson`
      const requestId = ++this.overlayRequestId

      try {
        const response = await fetch(assetUrl)
        if (!response.ok) {
          throw new Error(`Overlay request failed: ${response.status}`)
        }
        const payload = await response.json()
        if (requestId !== this.overlayRequestId) return
        this.overlayGeojson = payload
        this.selectedSnowLevel = level
      } catch (err) {
        this.error = err?.message || 'Unable to load overlay.'
      }
    },
    extractChance(text) {
      const match = text.match(/\s*([0-9]+)\s+percent/i)
      return match ? `${match[1]}%` : ''
    },
    extractChanceValue(text) {
      const match = text.match(/\s*([0-9]+)\s+percent/i)
      return match ? Number(match[1]) : null
    },
    async fetchForecast(preferredIndex = 0) {
      this.loading = true
      this.error = ''
      this.periods = []
      this.updated = ''
      this.geometry = null
      this.overlayGeojson = null
      this.selectedSnowLevel = ''
      this.selectedIndex = 0

      const url = `https://api.weather.gov/zones/forecast/${this.zoneId}/forecast`
      try {
        const response = await fetch(url, {
          headers: {
            Accept: 'application/geo+json',
            'User-Agent': 'cascade-forecast-viewer'
          }
        })
        if (!response.ok) {
          throw new Error(`Request failed: ${response.status}`)
        }
        const data = await response.json()
        this.periods = data?.properties?.periods || []
        this.updated = data?.properties?.updated || ''
        this.geometry = data?.geometry || null
        if (this.displayPeriods.length) {
          const safeIndex = Math.min(Math.max(preferredIndex, 0), this.displayPeriods.length - 1)
          this.selectPeriod(safeIndex)
        }
      } catch (err) {
        this.error = err?.message || 'Unable to load forecast data.'
      } finally {
        this.loading = false
      }
    }
  }
}
</script>

<style>
:root {
  --ink: #0f1f2e;
  --muted: #57707f;
  --accent: #829d99;
  --accent-2: #f4b860;
  --paper: #f7f3ec;
  --glass: rgba(255, 255, 255, 0.68);
  --shadow: 0 18px 40px rgba(15, 31, 46, 0.12);
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: 'Space Grotesk', 'Segoe UI', sans-serif;
  color: var(--ink);
  background: radial-gradient(circle at 10% 10%, #fef1d9, transparent 40%),
    radial-gradient(circle at 90% 20%, #d8f1ee, transparent 35%),
    radial-gradient(circle at 20% 80%, #dfe8ff, transparent 40%), var(--paper);
  min-height: 100vh;
}

.app {
  max-width: 1200px;
  margin: 0 auto;
  padding: 32px 20px 56px;
}

.header {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 26px;
}

.title-wrap {
  display: grid;
  gap: 6px;
}

h1 {
  font-family: 'Fraunces', 'Times New Roman', serif;
  font-size: clamp(2rem, 3vw, 3rem);
  margin: 0;
  letter-spacing: -0.02em;
}

.subtitle {
  color: var(--muted);
  font-size: 1rem;
  margin: 0;
}

.controls {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  align-items: center;
}

.zone-buttons {
  display: grid;
  gap: 10px;
}

.zone-buttons button {
  border: 1px solid transparent;
}

.zone-buttons button.active {
  background: #1f5c54;
  box-shadow: 0 14px 26px rgba(31, 92, 84, 0.28);
}

.input-group {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  background: var(--glass);
  border-radius: 999px;
  border: 1px solid rgba(15, 31, 46, 0.08);
  box-shadow: var(--shadow);
}

.input-group label {
  font-size: 0.85rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: var(--muted);
}

.input-group input {
  border: none;
  background: transparent;
  font-size: 1rem;
  min-width: 110px;
  outline: none;
  color: var(--ink);
  font-weight: 600;
  text-transform: uppercase;
}

button {
  border: none;
  background: var(--accent);
  color: #fff;
  padding: 12px 20px;
  border-radius: 999px;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 0 12px 20px rgba(42, 127, 116, 0.24);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

button:hover {
  transform: translateY(-2px);
  box-shadow: 0 14px 26px rgba(42, 127, 116, 0.28);
}

.grid {
  display: grid;
  grid-template-columns: minmax(0, 1.1fr) minmax(0, 0.9fr);
  gap: 24px;
}

.panel {
  background: var(--glass);
  border-radius: 22px;
  padding: 20px;
  border: 1px solid rgba(15, 31, 46, 0.08);
  box-shadow: var(--shadow);
  backdrop-filter: blur(16px);
}

.map-panel {
  display: flex;
  flex-direction: column;
  gap: 12px;
  height: 600px;
  overflow: hidden;
}

.forecast-panel {
  display: flex;
  flex-direction: column;
  gap: 16px;
  height: calc(100vh - 170px);
}

.meta {
  display: grid;
  gap: 8px;
  font-size: 0.95rem;
  color: var(--muted);
}

.meta strong {
  color: var(--ink);
  font-weight: 600;
}

.headline {
  font-family: 'Fraunces', 'Times New Roman', serif;
  font-size: 1.5rem;
  margin: 6px 0 10px;
}

.lead {
  margin: 0 0 10px;
  color: #2e3e4f;
  line-height: 1.45;
}

.badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: rgba(244, 184, 96, 0.2);
  color: #8a4f04;
  padding: 6px 12px;
  border-radius: 999px;
  font-size: 0.85rem;
  font-weight: 600;
}

.cards {
  display: grid;
  gap: 12px;
  margin-top: 16px;
  overflow-y: auto;
  padding-right: 6px;
}

.card {
  padding: 14px 16px;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.75);
  border: 1px solid rgba(15, 31, 46, 0.08);
  box-shadow: 0 10px 18px rgba(15, 31, 46, 0.08);
  animation: rise 0.6s ease both;
}

.card.is-clickable {
  cursor: pointer;
}

.card.active {
  border-color: rgba(106, 167, 255, 0.6);
  box-shadow: 0 12px 20px rgba(106, 167, 255, 0.2);
}

.card h3 {
  margin: 0 0 6px;
  font-size: 1.05rem;
}

.card p {
  margin: 0;
  color: #2e3e4f;
  line-height: 1.45;
}

.card .details {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  justify-content: space-between;
  margin-top: 10px;
  font-size: 0.85rem;
  color: var(--muted);
}

.status {
  font-size: 0.95rem;
  color: var(--muted);
}

.error {
  color: #b54434;
  background: rgba(181, 68, 52, 0.08);
  padding: 10px 12px;
  border-radius: 12px;
}

.loading-bar {
  height: 6px;
  border-radius: 999px;
  background: linear-gradient(90deg, #dfe8ff, #fef1d9, #d8f1ee);
  overflow: hidden;
  position: relative;
}

.loading-bar::after {
  content: '';
  position: absolute;
  inset: 0;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.8), transparent);
  animation: shimmer 1.4s ease infinite;
}

@keyframes shimmer {
  0% {
    transform: translateX(-100%);
  }

  100% {
    transform: translateX(100%);
  }
}

@keyframes rise {
  from {
    opacity: 0;
    transform: translateY(12px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@media (max-width: 980px) {
  .grid {
    grid-template-columns: 1fr;
  }

  .header {
    align-items: flex-start;
  }

  .forecast-panel {
    max-height: none;
  }
}

@media (max-width: 720px) {
  .app {
    position: static !important;
    inset: auto !important;
    overflow: visible !important;
    padding: 20px 16px 32px;
  }

  .header {
    gap: 12px;
  }

  .controls {
    width: 100%;
  }

  .zone-buttons {
    width: 100%;
  }

  .zone-buttons button {
    width: 100%;
  }

  .grid {
    gap: 18px;
  }

  .panel {
    padding: 16px;
  }

  .map-panel {
    height: 45vh;
    min-height: 260px;
    max-height: 420px;
  }

  .forecast-panel {
    height: auto;
  }

  .cards {
    margin-top: 10px;
    padding-right: 0;
    overflow: visible;
  }

  .card .details {
    justify-content: flex-start;
  }
}
</style>
