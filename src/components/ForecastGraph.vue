<template>
  <div v-if="graphMeta.ready"
       class="forecast-graph">
    <div class="forecast-graph__header">
      <span>Snow level (ft)</span>
      <span>Precip chance</span>
    </div>
    <div class="forecast-graph__body">
      <svg class="forecast-graph__chart"
           :viewBox="`0 0 ${graphMeta.width} ${graphMeta.height}`"
           role="img"
           aria-label="Snow level and precipitation chance forecast">
        <defs>
          <clipPath :id="clipId">
            <rect :x="graphMeta.paddingLeft"
                  :y="graphMeta.clipY"
                  :width="graphMeta.plotWidth"
                  :height="graphMeta.clipHeight"
                  rx="12" />
          </clipPath>
          <linearGradient id="snowLine"
                          x1="0"
                          y1="0"
                          x2="1"
                          y2="1">
            <stop offset="0%"
                  stop-color="#2a7f74" />
            <stop offset="100%"
                  stop-color="#1e4b6a" />
          </linearGradient>
          <linearGradient id="precipGlow"
                          x1="0"
                          y1="0"
                          x2="0"
                          y2="1">
            <stop offset="0%"
                  stop-color="rgba(60, 103, 155, 0.35)" />
            <stop offset="100%"
                  stop-color="rgba(60, 103, 155, 0.05)" />
          </linearGradient>
        </defs>
        <rect :x="graphMeta.paddingLeft"
              :y="graphMeta.paddingTop"
              :width="graphMeta.plotWidth"
              :height="graphMeta.plotHeight"
              class="forecast-graph__plot" />
        <g class="forecast-graph__grid">
          <line v-for="tick in graphMeta.snowTicks"
                :key="`grid-${tick.value}`"
                :x1="graphMeta.paddingLeft"
                :x2="graphMeta.paddingLeft + graphMeta.plotWidth"
                :y1="tick.y"
                :y2="tick.y"
                class="forecast-graph__grid-line" />
          <text v-for="tick in graphMeta.snowTicks"
                :key="`label-${tick.value}`"
                :x="graphMeta.paddingLeft - 10"
                :y="tick.y + 4"
                class="forecast-graph__grid-label">
            {{ tick.value }}
          </text>
        </g>
        <g :clip-path="`url(#${clipId})`">
          <path v-if="graphMeta.precipAreaPath"
                :d="graphMeta.precipAreaPath"
                class="forecast-graph__area" />
          <path v-if="graphMeta.precipPath"
                :d="graphMeta.precipPath"
                class="forecast-graph__line forecast-graph__line--precip" />
          <line v-if="graphMeta.highlightX !== null"
                :x1="graphMeta.highlightX"
                :x2="graphMeta.highlightX"
                :y1="graphMeta.paddingTop"
                :y2="graphMeta.paddingTop + graphMeta.plotHeight"
                class="forecast-graph__marker" />
          <g class="forecast-graph__points">
            <rect v-for="point in graphMeta.snowPointsFiltered"
                  :key="point.key"
                  :x="point.x - 10"
                  :y="point.y - 3"
                  width="20"
                  height="6"
                  rx="2"
                  class="forecast-graph__hatch" />
            <circle v-for="point in graphMeta.precipPointsFiltered"
                    :key="point.key"
                    :cx="point.x"
                    :cy="point.y"
                    r="3"
                    class="forecast-graph__dot forecast-graph__dot--precip" />
          </g>
        </g>
        <line :x1="graphMeta.paddingLeft"
              :x2="graphMeta.paddingLeft + graphMeta.plotWidth"
              :y1="graphMeta.paddingTop + graphMeta.plotHeight"
              :y2="graphMeta.paddingTop + graphMeta.plotHeight"
              class="forecast-graph__baseline" />
      </svg>
      <div class="forecast-graph__y-axis forecast-graph__y-axis--right">
        <span>100%</span>
        <span>0%</span>
      </div>
    </div>
    <div class="forecast-graph__labels"
         :style="{ gridTemplateColumns: `repeat(${graphMeta.labels.length}, minmax(0, 1fr))` }">
      <span v-for="(label, index) in graphMeta.labels"
            :key="`label-${index}`">
        {{ label }}
      </span>
    </div>
    <div v-if="sliderCount > 1"
         class="forecast-graph__slider">
      <div class="forecast-graph__slider-meta">
        <div class="forecast-graph__slider-label">
          <strong>{{ sliderLabel || `Day ${sliderIndex + 1}` }}</strong>
        </div>
        <div v-if="snowLevelLabel"
             class="forecast-graph__slider-label">
          <strong>{{ snowLevelLabel }}</strong>
        </div>
      </div>
      <input type="range"
             min="0"
             :max="sliderCount - 1"
             step="1"
             :value="sliderIndex"
             @input="handleSliderInput" />
    </div>
  </div>
</template>

<script>
import { computed } from 'vue'

export default {
  name: 'ForecastGraph',
  props: {
    graphPeriods: {
      type: Array,
      default: () => []
    },
    sliderCount: {
      type: Number,
      default: 0
    },
    sliderIndex: {
      type: Number,
      default: 0
    },
    sliderLabel: {
      type: String,
      default: ''
    },
    snowLevel: {
      type: [String, Number],
      default: ''
    }
  },
  emits: ['update:sliderIndex'],
  setup(props, { emit }) {
    const clipId = `plot-clip-${Math.random().toString(36).slice(2, 8)}`

    const handleSliderInput = (event) => {
      const nextValue = Number(event.target.value)
      emit('update:sliderIndex', Number.isNaN(nextValue) ? 0 : nextValue)
    }

    const snowLevelLabel = computed(() => {
      if (props.snowLevel === null || props.snowLevel === undefined) return ''
      const raw = String(props.snowLevel).trim()
      if (!raw) return ''
      return /ft$/i.test(raw) ? raw : `${raw} ft`
    })

    const graphMeta = computed(() => {
      const width = 640
      const height = 210
      const paddingLeft = 60
      const paddingRight = 60
      const paddingTop = 18
      const paddingBottom = 30
      const plotWidth = width - paddingLeft - paddingRight
      const plotHeight = height - paddingTop - paddingBottom

      const periods = Array.isArray(props.graphPeriods) ? props.graphPeriods : []
      const labels = periods.map((period, index) => period?.name || `Day ${index + 1}`)
      const toNumber = (value) => {
        const num = Number(value)
        return Number.isFinite(num) ? num : null
      }
      const snowValues = periods.map((period) => toNumber(period?.snowLevel))
      const precipValues = periods.map((period) => toNumber(period?.precipChance - 10))
      const count = periods.length
      const xForIndex = (index) => {
        if (count <= 1) return paddingLeft + plotWidth / 2
        return paddingLeft + (index / (count - 1)) * plotWidth
      }
      const yForValue = (value, maxValue) => {
        const normalized = Math.min(Math.max(value / maxValue, 0), 1)
        return paddingTop + plotHeight - normalized * plotHeight
      }
      const maxSnowRaw = Math.max(0, ...snowValues.filter((value) => value !== null))
      const snowStep = 1000
      const snowMax = maxSnowRaw > 0 ? Math.ceil(maxSnowRaw / snowStep) * snowStep : snowStep
      const snowTicks = []
      for (let value = 0; value <= snowMax; value += snowStep) {
        snowTicks.push({ value, y: yForValue(value, snowMax) })
      }
      const snowPoints = snowValues.map((value, index) => {
        if (value === null) return null
        return { x: xForIndex(index), y: yForValue(value, snowMax), value, key: `snow-${index}` }
      })
      const precipPoints = precipValues.map((value, index) => {
        if (value === null) return null
        return { x: xForIndex(index), y: yForValue(value, 100), value, key: `precip-${index}` }
      })
      const buildSmoothSegment = (segment) => {
        if (!segment.length) return ''
        if (segment.length === 1) {
          const point = segment[0]
          return `M ${point.x} ${point.y}`
        }
        const smoothing = 0.2
        let path = `M ${segment[0].x} ${segment[0].y} `
        for (let i = 0; i < segment.length - 1; i += 1) {
          const p0 = segment[i - 1] || segment[i]
          const p1 = segment[i]
          const p2 = segment[i + 1]
          const p3 = segment[i + 2] || p2
          const cp1x = p1.x + (p2.x - p0.x) * smoothing
          const cp1y = p1.y + (p2.y - p0.y) * smoothing
          const cp2x = p2.x - (p3.x - p1.x) * smoothing
          const cp2y = p2.y - (p3.y - p1.y) * smoothing
          path += `C ${cp1x} ${cp1y} ${cp2x} ${cp2y} ${p2.x} ${p2.y} `
        }
        return path.trim()
      }
      const buildLinePath = (points) => {
        let path = ''
        let segment = []
        points.forEach((point) => {
          if (!point) {
            if (segment.length) {
              path += `${path ? ' ' : ''}${buildSmoothSegment(segment)}`
              segment = []
            }
            return
          }
          segment.push(point)
        })
        if (segment.length) {
          path += `${path ? ' ' : ''}${buildSmoothSegment(segment)}`
        }
        return path.trim()
      }
      const buildAreaPath = (points) => {
        if (!points.length || points.some((point) => !point)) return ''
        const start = points[0]
        const end = points[points.length - 1]
        let path = `${buildSmoothSegment(points)} `
        path += `L ${end.x} ${paddingTop + plotHeight} `
        path += `L ${start.x} ${paddingTop + plotHeight} Z`
        return path.trim()
      }

      return {
        width,
        height,
        paddingLeft,
        paddingTop,
        plotWidth,
        plotHeight,
        labels,
        snowTicks,
        snowPointsFiltered: snowPoints.filter(Boolean),
        precipPointsFiltered: precipPoints.filter(Boolean),
        precipPath: buildLinePath(precipPoints),
        precipAreaPath: buildAreaPath(precipPoints),
        highlightX: count ? xForIndex(Math.min(Math.max(props.sliderIndex, 0), count - 1)) : null,
        clipY: Math.max(0, paddingTop - 10),
        clipHeight: plotHeight + 10,
        ready: periods.length > 0
      }
    })

    return {
      clipId,
      graphMeta,
      handleSliderInput,
      snowLevelLabel
    }
  }
}
</script>

<style scoped>
.forecast-graph {
  display: grid;
  /* gap: 12px;
  padding: 14px 16px 12px; */
  background: linear-gradient(130deg, rgba(255, 255, 255, 0.95), rgba(235, 244, 250, 0.95));
  border-radius: 18px;
  border: 1px solid rgba(15, 31, 46, 0.08);
}

.forecast-graph__header {
  display: flex;
  justify-content: space-between;
  font-size: 0.7rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  padding: 12px 16px 0;
  color: rgba(46, 62, 79, 0.65);
}

.forecast-graph__body {
  position: relative;
}

.forecast-graph__chart {
  width: 100%;
  height: 200px;
  display: block;
  margin: -20px 0 -34px
}

.forecast-graph__plot {
  fill: rgba(255, 255, 255, 0.6);
  stroke: rgba(46, 62, 79, 0.08);
  stroke-width: 1;
  rx: 12;
}

.forecast-graph__grid-line {
  stroke: rgba(46, 62, 79, 0.08);
  stroke-width: 1;
}

.forecast-graph__grid-label {
  fill: rgba(46, 62, 79, 0.6);
  font-size: 11px;
  text-anchor: end;
}

.forecast-graph__line {
  fill: none;
  stroke-width: 3;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.forecast-graph__line--precip {
  stroke: #3c679b;
  stroke-dasharray: 6 6;
}

.forecast-graph__area {
  fill: url(#precipGlow);
}

.forecast-graph__marker {
  stroke: rgba(46, 62, 79, 0.35);
  stroke-width: 1.5;
  stroke-dasharray: 3 4;
}

.forecast-graph__baseline {
  stroke: rgba(46, 62, 79, 0.18);
  stroke-width: 1;
}

.forecast-graph__dot {
  stroke: rgba(255, 255, 255, 0.9);
  stroke-width: 1;
}

.forecast-graph__dot--precip {
  fill: #3c679b;
}

.forecast-graph__hatch {
  fill: #2a7f74;
}

.forecast-graph__y-axis {
  position: absolute;
  top: 10px;
  bottom: 5px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  font-size: 0.7rem;
  color: rgba(46, 62, 79, 0.6);
}

.forecast-graph__y-axis--right {
  right: 6px;
  text-align: center;
  margin-right: 5px
}

.forecast-graph__labels {
  display: grid;
  gap: 8px;
  font-size: 0.75rem;
  color: rgba(46, 62, 79, 0.75);
  text-align: center;
}

.forecast-graph__slider {
  display: grid;
  gap: 10px;
  padding: 10px 10px 20px 10px;
}

.forecast-graph__slider-meta {
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 8px 18px;
}

.forecast-graph__slider-label {
  display: grid;
  gap: 4px;
  font-size: 0.7rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: rgba(46, 62, 79, 0.7);
}

.forecast-graph__slider-label strong {
  font-size: 0.9rem;
  letter-spacing: 0;
  text-transform: none;
  color: #2e3e4f;
  font-weight: 600;
}

.forecast-graph__slider input[type='range'] {
  width: 100%;
  accent-color: #2a7f74;
}

@media (max-width: 980px) {
  .forecast-graph__chart {
    height: 170px;
  }
}

@media (max-width: 720px) {
  .forecast-graph__header {
    padding: 10px 12px 0;
    font-size: 0.65rem;
  }

  .forecast-graph__chart {
    height: 150px;
    margin: -16px 0 -28px;
  }

  .forecast-graph__labels {
    gap: 6px;
    font-size: 0.68rem;
  }

  .forecast-graph__slider {
    padding: 8px 10px 16px 10px;
  }

  .forecast-graph__slider-meta {
    flex-direction: column;
    align-items: flex-start;
  }
}
</style>
