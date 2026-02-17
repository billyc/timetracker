<script setup lang="ts">
import { ref, computed, onMounted, nextTick } from 'vue'
import { Bar } from 'vue-chartjs'
import { Chart as ChartJS, BarElement, CategoryScale, LinearScale, Tooltip } from 'chart.js'

ChartJS.register(BarElement, CategoryScale, LinearScale, Tooltip)

interface TimeTrackerEntry {
  date: string
  value: number
  note?: string
}

interface HeatmapCell {
  date: string
  day: number
  minutes: number
  inMonth: boolean
}

const STORAGE_KEY = 'timetracker'

const entries = ref<TimeTrackerEntry[]>([])
const date = ref(new Date().toISOString().slice(0, 10))
const minutes = ref<number | null>(null)
const note = ref('')

function load() {
  const raw = localStorage.getItem(STORAGE_KEY)
  if (raw) {
    entries.value = JSON.parse(raw) as TimeTrackerEntry[]
  }
}

function save() {
  if (minutes.value == null) return
  entries.value.push({
    date: date.value,
    value: minutes.value,
    note: note.value || undefined,
  })
  localStorage.setItem(STORAGE_KEY, JSON.stringify(entries.value))
  minutes.value = null
  note.value = ''
}

const sortedForTable = computed(() =>
  [...entries.value].sort((a, b) => b.date.localeCompare(a.date))
)

const dailyTotals = computed(() => {
  const map = new Map<string, number>()
  for (const e of entries.value) {
    map.set(e.date, (map.get(e.date) ?? 0) + e.value)
  }
  return map
})

interface HeatmapMonth {
  key: string
  label: string
  weeks: HeatmapCell[][]
}

function weeksForMonth(year: number, month: number): HeatmapCell[][] {
  const firstOfMonth = new Date(year, month - 1, 1)
  const lastOfMonth = new Date(year, month, 0)

  const start = new Date(firstOfMonth)
  start.setDate(start.getDate() - start.getDay())

  const end = new Date(lastOfMonth)
  if (end.getDay() < 6) end.setDate(end.getDate() + (6 - end.getDay()))

  const weeks: HeatmapCell[][] = []
  const cursor = new Date(start)
  while (cursor <= end) {
    const week: HeatmapCell[] = []
    for (let d = 0; d < 7; d++) {
      const iso = `${cursor.getFullYear()}-${String(cursor.getMonth() + 1).padStart(2, '0')}-${String(cursor.getDate()).padStart(2, '0')}`
      week.push({
        date: iso,
        day: cursor.getDate(),
        minutes: dailyTotals.value.get(iso) ?? 0,
        inMonth: cursor.getMonth() === month - 1,
      })
      cursor.setDate(cursor.getDate() + 1)
    }
    weeks.push(week)
  }
  return weeks
}

const heatmapMonths = computed(() => {
  const now = new Date()
  const currentMonth = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`
  const monthSet = new Set<string>([currentMonth])
  for (const e of entries.value) {
    monthSet.add(e.date.slice(0, 7))
  }
  const sorted = [...monthSet].sort().reverse()

  return sorted.map((key): HeatmapMonth => {
    const [y, m] = key.split('-').map(Number) as [number, number]
    const weeks = weeksForMonth(y, m)
    const label = new Date(y, m - 1).toLocaleString('default', { month: 'long', year: 'numeric' })
    return { key, label, weeks }
  })
})

const colorStops: { min: number; color: string }[] = [
  { min: 0, color: 'transparent' },
  { min: 1, color: '#d6c65B' },
  { min: 15, color: '#68daaa' },
  { min: 30, color: '#53a8d4' },
  { min: 60, color: '#5064e0' },
  { min: 90, color: '#623ea9' },
  { min: 120, color: '#cd4c9e' },
]

function cellColor(cell: HeatmapCell): string {
  let color = colorStops[0]!.color
  for (const stop of colorStops) {
    if (cell.minutes >= stop.min) color = stop.color
  }
  return color
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
  const rows = [
    'date,minutes,note',
    ...entries.value.map(e => `${e.date},${e.value},"${(e.note ?? '').replace(/"/g, '""')}"`),
  ]
  const blob = new Blob([rows.join('\n')], { type: 'text/csv' })
  download(blob, 'timetracker.csv')
}

const editingDate = ref<string | null>(null)
const editingValue = ref<number | null>(null)
const editInput = ref<HTMLInputElement | null>(null)

function tapCell(cellDate: string, currentMinutes: number) {
  editingDate.value = cellDate
  editingValue.value = currentMinutes || null
  nextTick(() => {
    editInput.value?.focus()
    editInput.value?.select()
  })
}

function commitEdit() {
  const cellDate = editingDate.value
  if (!cellDate) return
  const val = editingValue.value

  entries.value = entries.value.filter(e => e.date !== cellDate)

  if (val != null && val > 0) {
    entries.value.push({ date: cellDate, value: val })
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(entries.value))
  editingDate.value = null
  editingValue.value = null
}

const totalHours = computed(() => {
  const totalMin = entries.value.reduce((sum, e) => sum + e.value, 0)
  return (totalMin / 60).toFixed(1)
})

function isoWeekLabel(dateStr: string): string {
  const d = new Date(dateStr + 'T00:00:00')
  const day = d.getDay()
  const thu = new Date(d)
  thu.setDate(d.getDate() - ((day + 6) % 7) + 3)
  const yearStart = new Date(thu.getFullYear(), 0, 1)
  const weekNum = Math.ceil(((thu.getTime() - yearStart.getTime()) / 86400000 + 1) / 7)
  return `${thu.getFullYear()}-W${String(weekNum).padStart(2, '0')}`
}

const weeklyChartData = computed(() => {
  const map = new Map<string, number>()
  for (const e of entries.value) {
    const week = isoWeekLabel(e.date)
    map.set(week, (map.get(week) ?? 0) + e.value)
  }
  if (map.size === 0) return { labels: [], datasets: [] }

  const weeks = [...map.keys()].sort()
  const firstWeek = weeks[0]!
  const lastWeek = weeks[weeks.length - 1]!

  // Fill in all weeks between first and last (including gaps with 0)
  const allWeeks: string[] = []
  const [fy, fw] = firstWeek.split('-W').map(Number) as [number, number]
  const [ly, lw] = lastWeek.split('-W').map(Number) as [number, number]
  let cy = fy
  let cw = fw
  while (cy < ly || (cy === ly && cw <= lw)) {
    allWeeks.push(`${cy}-W${String(cw).padStart(2, '0')}`)
    cw++
    // ISO weeks max out at 52 or 53; approximate with 52 then let label logic handle it
    const lastDay = new Date(cy, 11, 31)
    const maxWeek = isoWeekLabel(lastDay.toISOString().slice(0, 10)).split('-W').map(Number)[1]!
    if (cw > maxWeek) {
      cw = 1
      cy++
    }
  }

  const recent = allWeeks.slice(-12)
  const weeklyData = recent.map(w => map.get(w) ?? 0)
  const priorCumulative: number[] = []
  let runningTotal = 0
  for (const v of weeklyData) {
    priorCumulative.push(runningTotal)
    runningTotal += v
  }
  // Build x-axis labels: show month name on the first week of each month
  let lastMonth = ''
  const xLabels = recent.map(w => {
    const [wy, ww] = w.split('-W').map(Number) as [number, number]
    // Find the Monday of this ISO week
    const jan4 = new Date(wy, 0, 4)
    const mon = new Date(jan4)
    mon.setDate(jan4.getDate() - ((jan4.getDay() + 6) % 7) + (ww - 1) * 7)
    const monthKey = `${mon.getFullYear()}-${mon.getMonth()}`
    if (monthKey !== lastMonth) {
      lastMonth = monthKey
      if (mon.getMonth() === 0) {
        return mon.toLocaleString('default', { month: 'short' }) + ' ' + mon.getFullYear()
      }
      return mon.toLocaleString('default', { month: 'short' })
    }
    return ''
  })

  return {
    labels: xLabels,
    datasets: [
      {
        label: 'Cumulative',
        data: priorCumulative,
        backgroundColor: '#646cff',
        stack: 'stack',
      },
      {
        label: 'This Week',
        data: weeklyData,
        backgroundColor: '#83d583',
        stack: 'stack',
      },
    ],
  }
})

const weeklyChartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  barPercentage: 0.98,
  categoryPercentage: 1.0,
  plugins: { legend: { display: false } },
  scales: {
    x: { stacked: true, ticks: { color: '#888', font: { size: 10 } }, grid: { display: false } },
    y: {
      stacked: true,
      ticks: {
        color: '#888',
        stepSize: 60,
        callback: (v: number | string) => Math.round(Number(v) / 60) + 'h',
      },
      grid: { color: 'rgba(255,255,255,0.06)' },
    },
  },
}

function cancelEdit() {
  editingDate.value = null
  editingValue.value = null
}

function removeEntry(entry: TimeTrackerEntry) {
  const idx = entries.value.indexOf(entry)
  if (idx !== -1) {
    entries.value.splice(idx, 1)
    localStorage.setItem(STORAGE_KEY, JSON.stringify(entries.value))
  }
}

const initialLoad = ref(true)

function cellAnimDelay(mi: number, wi: number, ci: number): number {
  let offset = 0
  for (let i = 0; i < mi; i++) {
    offset += heatmapMonths.value[i]!.weeks.length * 7
  }
  return (offset + wi * 7 + ci) * 10
}

onMounted(() => {
  load()
  const totalCells = heatmapMonths.value.reduce((sum, m) => sum + m.weeks.length * 7, 0)
  setTimeout(
    () => {
      initialLoad.value = false
    },
    totalCells * 10 + 150
  )
})
</script>

<template>
  <h1>Time Tracker</h1>
  <div v-if="entries.length" class="total-banner">
    <span style="color: #83d583">{{ totalHours }}</span> hours total
  </div>

  <form class="entry-form" @submit.prevent="save">
    <div class="entry-row">
      <input type="date" v-model="date" required />
      <input type="number" v-model.number="minutes" step="1" placeholder="minutes" required />
      <button type="submit">+</button>
    </div>
    <input type="text" v-model="note" placeholder="notes" class="note-input" />
  </form>

  <div v-if="entries.length" class="chart-container">
    <div class="heatmap-title">Cumulative hours of input by week</div>
    <div class="chart-wrap">
      <Bar :data="weeklyChartData" :options="weeklyChartOptions" />
    </div>
  </div>

  <div v-for="(month, mi) in heatmapMonths" :key="month.key" class="heatmap-container">
    <div class="heatmap-title">{{ month.label }}</div>
    <table class="heatmap">
      <thead>
        <tr>
          <th v-for="d in ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat']" :key="d">{{ d }}</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(week, wi) in month.weeks" :key="wi">
          <td
            v-for="(cell, ci) in week"
            :key="cell.date"
            class="heatmap-cell"
            :class="{ 'out-of-month': !cell.inMonth, 'cell-animate': initialLoad }"
            :style="{
              backgroundColor: cellColor(cell),
              animationDelay: initialLoad ? cellAnimDelay(mi, wi, ci) + 'ms' : undefined,
            }"
            @click="tapCell(cell.date, cell.minutes)"
          >
            <span class="heatmap-day">{{ cell.day }}</span>
            <span v-if="cell.minutes" class="heatmap-minutes">{{ cell.minutes }}'</span>
          </td>
        </tr>
      </tbody>
    </table>
  </div>

  <div v-if="editingDate" class="modal-overlay" @click.self="cancelEdit">
    <div class="modal">
      <div class="modal-title">Edit total minutes for {{ editingDate }}</div>
      <input
        ref="editInput"
        type="number"
        inputmode="numeric"
        class="modal-input"
        v-model.number="editingValue"
        placeholder="Minutes"
        @keydown.enter="commitEdit"
        @keydown.escape="cancelEdit"
      />
      <div class="modal-buttons">
        <button type="button" @click="cancelEdit">Cancel</button>
        <button type="button" @click="commitEdit">Save</button>
      </div>
    </div>
  </div>

  <div class="heatmap-title">Data entries</div>
  <table v-if="entries.length" class="data-table">
    <thead>
      <tr>
        <th>Date</th>
        <th>Min</th>
        <th>Note</th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="entry in sortedForTable" :key="entry.date + entry.value">
        <td>{{ entry.date }}</td>
        <td>{{ entry.value }}</td>
        <td class="note-cell">{{ entry.note ?? '' }}</td>
        <td class="delete-cell">
          <span class="delete-btn" @click="removeEntry(entry)">&times;</span>
        </td>
      </tr>
    </tbody>
  </table>

  <p v-if="entries.length" class="export">
    Export data as:&nbsp;
    <a href="#" @click.prevent="exportCSV">CSV</a>&nbsp;
    <a href="#" @click.prevent="exportJSON">JSON</a>
  </p>
</template>

<style scoped>
h1 {
  margin: 0 0;
}

.total-banner {
  color: #36c;
  font-size: 1.2rem;
  font-weight: bold;
  padding: 0.15em 1em;
  margin-bottom: 1rem;
}

.entry-form {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 2rem;
  max-width: 400px;
  margin-left: auto;
  margin-right: auto;
}

.entry-row {
  display: flex;
  gap: 0.5rem;
}

.entry-row input[type='date'] {
  flex: 1;
  min-width: 0;
}

.entry-row input[type='number'] {
  width: 5rem;
}

.entry-form input {
  padding: 0.5em 0.75em;
  border-radius: 8px;
  border: 1px solid #444;
  background: #1a1a1a;
  color: inherit;
  font-size: 1em;
}

.note-input {
  width: 100%;
  box-sizing: border-box;
}

.heatmap-container {
  max-width: 400px;
  margin: 0 auto 1rem;
}

.heatmap-title {
  font-weight: 600;
  margin-bottom: 0rem;
  text-align: left;
}

.heatmap {
  width: 100%;
  border-collapse: collapse;
  table-layout: fixed;
}

.heatmap th {
  font-size: 0.7em;
  color: #888;
  padding: 0.25em 0;
  font-weight: 500;
}

.heatmap-cell {
  position: relative;
  height: 2rem;
  vertical-align: top;
  border: 2px solid #1a1a1a;
  border-radius: 4px;
  padding: 2px;
  text-align: center;
}

.heatmap-cell.out-of-month {
  opacity: 0.3;
}

@keyframes cellFadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.cell-animate {
  animation: cellFadeIn 0.15s linear backwards;
}

.heatmap-day {
  display: block;
  font-size: 0.65em;
  color: #555;
  line-height: 1;
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: flex-start;
  padding-top: 25vh;
  justify-content: center;
  z-index: 100;
}

.modal {
  background: #2a2a2a;
  border-radius: 12px;
  padding: 1.25rem;
  min-width: 240px;
  text-align: center;
}

.modal-title {
  font-size: 0.9em;
  color: #aaa;
  margin-bottom: 0.75rem;
}

.modal-input {
  width: 100%;
  box-sizing: border-box;
  padding: 0.5em 0.75em;
  border-radius: 8px;
  border: 1px solid #444;
  background: #1a1a1a;
  color: inherit;
  font-size: 1.1em;
  text-align: center;
  margin-bottom: 0.75rem;
}

.modal-buttons {
  display: flex;
  gap: 0.5rem;
}

.modal-buttons button {
  flex: 1;
  padding: 0.5em;
  font-size: 0.9em;
}

.heatmap-minutes {
  display: block;
  margin-top: 0.25rem;
  font-size: 0.75em;
  font-weight: 700;
  color: #fff;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.6);
  line-height: 1;
}

.chart-container {
  max-width: 400px;
  margin: 0 auto 1.5rem;
}

.chart-wrap {
  height: 180px;
}

.data-table {
  font-size: 13px;
  line-height: 1.2;
}

.data-table td,
.data-table th {
  vertical-align: top;
  padding: 0.25em 0.25em;
}

.note-cell {
  text-align: left;
  word-break: break-word;
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
  cursor: pointer;
  color: #bbb;
  font-size: 1.1em;
}

.delete-btn:hover {
  color: #e53e3e;
}

@media (prefers-color-scheme: light) {
  .entry-form input {
    background: #f9f9f9;
    border-color: #ccc;
  }
  .heatmap-cell {
    border-color: #f9f9f9;
  }
  .modal {
    background: #fff;
  }
  .modal-input {
    background: #f9f9f9;
    border-color: #ccc;
  }
}
</style>
