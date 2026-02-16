<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

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
  const monthSet = new Set<string>()
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
  { min: 1, color: '#8bf' },
  { min: 15, color: '#4c7' },
  { min: 30, color: '#db4' },
  { min: 60, color: 'darkorange' },
  { min: 90, color: '#c00' },
  { min: 120, color: '#92b' },
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

function tapCell(cellDate: string, currentMinutes: number) {
  const input = prompt(
    `Minutes for ${cellDate}:`,
    currentMinutes ? String(currentMinutes) : '',
  )
  if (input == null) return
  const val = parseInt(input, 10)
  if (isNaN(val)) return

  // Remove all existing entries for this date
  entries.value = entries.value.filter(e => e.date !== cellDate)

  // Add new entry if value > 0
  if (val > 0) {
    entries.value.push({ date: cellDate, value: val })
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(entries.value))
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
  <h1>Time Tracker</h1>

  <form class="entry-form" @submit.prevent="save">
    <div class="entry-row">
      <input type="date" v-model="date" required />
      <input type="number" v-model.number="minutes" step="1" placeholder="Minutes" required />
      <button type="submit">+</button>
    </div>
    <input type="text" v-model="note" placeholder="Note (optional)" class="note-input" />
  </form>

  <div v-for="month in heatmapMonths" :key="month.key" class="heatmap-container">
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
            v-for="cell in week"
            :key="cell.date"
            class="heatmap-cell"
            :class="{ 'out-of-month': !cell.inMonth }"
            :style="{ backgroundColor: cellColor(cell) }"
            @click="tapCell(cell.date, cell.minutes)"
          >
            <span class="heatmap-day">{{ cell.day }}</span>
            <span v-if="cell.minutes" class="heatmap-minutes">{{ cell.minutes }}'</span>
          </td>
        </tr>
      </tbody>
    </table>
  </div>

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
  margin: 0 auto 2rem;
}

.heatmap-title {
  font-weight: 600;
  font-size: 1.05em;
  text-align: center;
  margin-bottom: 0.5rem;
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

.heatmap-day {
  display: block;
  font-size: 0.65em;
  color: #555;
  line-height: 1;
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
}
</style>
