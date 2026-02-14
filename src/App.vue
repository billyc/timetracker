<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { Line } from 'vue-chartjs'
import {
  Chart as ChartJS,
  type Chart,
  type ChartOptions,
  type Plugin,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
} from 'chart.js'

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip)

interface TimeTrackerEntry {
  date: string
  value: number
}

const STORAGE_KEY = 'timetracker'

const entries = ref<TimeTrackerEntry[]>([])
const date = ref(new Date().toISOString().slice(0, 10))
const weight = ref<number | null>(null)

function load() {
  const raw = localStorage.getItem(STORAGE_KEY)
  if (raw) {
    entries.value = JSON.parse(raw) as TimeTrackerEntry[]
  }
}

function save() {
  if (weight.value == null) return
  entries.value.push({ date: date.value, value: weight.value })
  localStorage.setItem(STORAGE_KEY, JSON.stringify(entries.value))
  weight.value = null
}

const sortedForChart = computed(() =>
  [...entries.value].sort((a, b) => a.date.localeCompare(b.date))
)

const sortedForTable = computed(() =>
  [...entries.value].sort((a, b) => b.date.localeCompare(a.date))
)

const trendData = computed(() => {
  const values = sortedForChart.value.map(e => e.value)
  const trend: number[] = []
  for (let i = 0; i < values.length; i++) {
    if (i === 0) {
      trend.push(values[0]!)
    } else {
      trend.push(trend[i - 1]! + 0.12 * (values[i]! - trend[i - 1]!))
    }
  }
  return trend
})

const chartData = computed(() => ({
  labels: sortedForChart.value.map(e => e.date),
  datasets: [
    {
      label: 'TimeTracker',
      data: sortedForChart.value.map(e => e.value),
      borderColor: 'transparent',
      backgroundColor: '#22c55e',
      pointRadius: 3,
      showLine: false,
    },
    {
      label: 'Trend',
      data: trendData.value,
      borderColor: '#3b82f6',
      backgroundColor: '#3b82f6',
      tension: 0.3,
      pointRadius: 0,
      borderWidth: 3,
    },
  ],
}))

const diffLinesPlugin: Plugin<'line'> = {
  id: 'diffLines',
  afterDatasetsDraw(chart: Chart<'line'>) {
    const weightMeta = chart.getDatasetMeta(0)
    const trendMeta = chart.getDatasetMeta(1)
    if (!weightMeta || !trendMeta) return
    const ctx = chart.ctx
    ctx.save()
    ctx.strokeStyle = 'rgba(249, 115, 22, 0.5)'
    ctx.lineWidth = 1.5
    for (let i = 0; i < weightMeta.data.length; i++) {
      const wp = weightMeta.data[i]
      const tp = trendMeta.data[i]
      if (!wp || !tp) continue
      ctx.beginPath()
      ctx.moveTo(wp.x, wp.y)
      ctx.lineTo(tp.x, tp.y)
      ctx.stroke()
    }
    ctx.restore()
  },
}

const chartOptions: ChartOptions<'line'> = {
  responsive: true,
  plugins: { title: { display: false } },
  scales: {
    x: { ticks: { color: '#aaa' }, grid: { display: false }, border: { color: '#555' } },
    y: {
      ticks: { color: '#aaa' },
      grid: { display: false },
      border: { color: '#555' },
      grace: '15%',
    },
  },
}

function download(blob: Blob, filename: string) {
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename
  a.click()
  URL.revokeObjectURL(url)
}

function exportJSON() {
  const blob = new Blob([JSON.stringify(entries.value, null, 2)], {
    type: 'application/json',
  })
  download(blob, 'timetracker.json')
}

function exportCSV() {
  const rows = ['date,time', ...entries.value.map(e => `${e.date},${e.value}`)]
  const blob = new Blob([rows.join('\n')], { type: 'text/csv' })
  download(blob, 'timetracker.csv')
}

function removeEntry(entry: TimeTrackerEntry) {
  const idx = entries.value.indexOf(entry)
  if (idx !== -1) {
    entries.value.splice(idx, 1)
    localStorage.setItem(STORAGE_KEY, JSON.stringify(entries.value))
  }
}

onMounted(load)
</script>

<template>
  <h1>Time Series Data Tracker</h1>

  <form class="entry-form" @submit.prevent="save">
    <input type="date" v-model="date" required />
    <input type="number" v-model.number="weight" step="0.1" placeholder="minutes" required />
    <button type="submit">Save</button>
  </form>

  <div v-if="entries.length" class="chart-container">
    <Line :data="chartData" :options="chartOptions" :plugins="[diffLinesPlugin]" />
  </div>

  <table v-if="entries.length">
    <thead>
      <tr>
        <th>Date</th>
        <th>Minutes</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="entry in sortedForTable" :key="entry.date + entry.value">
        <td>{{ entry.date }}</td>
        <td>{{ entry.value }}</td>
        <td class="delete-cell">
          <span class="delete-btn" @click="removeEntry(entry)">&times;</span>
        </td>
      </tr>
    </tbody>
  </table>

  <p v-if="entries.length" class="export">
    Export data as: <a href="#" @click.prevent="exportJSON">JSON</a>&nbsp;
    <a href="#" @click.prevent="exportCSV">CSV</a>
  </p>
</template>

<style scoped>
.entry-form {
  display: flex;
  gap: 0.5rem;
  justify-content: center;
  margin-bottom: 2rem;
}

.entry-form input {
  padding: 0.5em 0.75em;
  border-radius: 8px;
  border: 1px solid #444;
  background: #1a1a1a;
  color: inherit;
  font-size: 1em;
}

.chart-container {
  max-width: 700px;
  margin: 0 auto 2rem;
}

.export {
  margin-top: 1.5rem;
  font-size: 0.85em;
  color: #888;
}

.export a {
  color: #646cff;
}

.delete-cell {
  border: none;
  width: 1.5em;
  padding: 0;
}

.delete-btn {
  visibility: hidden;
  cursor: pointer;
  color: #888;
  font-size: 1.1em;
}

tr:hover .delete-btn {
  visibility: visible;
}

.delete-btn:hover {
  color: #e53e3e;
}

@media (prefers-color-scheme: light) {
  .entry-form input {
    background: #f9f9f9;
    border-color: #ccc;
  }
}
</style>
