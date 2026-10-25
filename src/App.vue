<script setup lang="ts">
import { ref, computed, onMounted, nextTick } from 'vue'
import { Bar } from 'vue-chartjs'
import { Chart as ChartJS, BarElement, CategoryScale, LinearScale, Tooltip } from 'chart.js'

ChartJS.register(BarElement, CategoryScale, LinearScale, Tooltip)

type EntryType = 'input' | 'talking' | 'reading'
const ENTRY_TYPES: EntryType[] = ['input', 'talking', 'reading']
const TYPE_COLORS = { talking: '#e8c547', reading: '#d47a5e' }
const INPUT_COLORS = { cumulative: '#646cff', thisWeek: '#83d583' }
const BUTTON_COLORS: Record<EntryType, string> = { input: INPUT_COLORS.cumulative, ...TYPE_COLORS }

interface TimeTrackerEntry {
  date: string
  value: number
  type: EntryType
  note?: string
}

type TypeTotals = Record<EntryType, number>

interface HeatmapCell {
  date: string
  day: number
  minutes: number
  inMonth: boolean
}

const STORAGE_KEY = 'timetracker'
const INITIAL_HOURS_KEY = 'timetracker_initial_hours'
const ENTRY_TYPE_KEY = 'timetracker_entry_type'

const entries = ref<TimeTrackerEntry[]>([])
const date = ref(new Date().toISOString().slice(0, 10))
const minutes = ref<number | null>(null)
const note = ref('')
const initialHours = ref<number | null>(null)
const entryType = ref<EntryType>('input')

function isEntryType(t: unknown): t is EntryType {
  return ENTRY_TYPES.includes(t as EntryType)
}

// Older data has no "type" field; those entries are all input
function withTypes(data: Omit<TimeTrackerEntry, 'type'>[]): TimeTrackerEntry[] {
  return data.map(e => {
    const type = (e as Partial<TimeTrackerEntry>).type
    return isEntryType(type) ? (e as TimeTrackerEntry) : { ...e, type: 'input' }
  })
}

function persist() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(entries.value))
}

function load() {
  const raw = localStorage.getItem(STORAGE_KEY)
  if (raw) {
    const stored = JSON.parse(raw) as TimeTrackerEntry[]
    const migrated = withTypes(stored)
    entries.value = migrated
    if (migrated.some((e, i) => e !== stored[i])) persist()
  }
  const savedHours = localStorage.getItem(INITIAL_HOURS_KEY)
  if (savedHours) {
    initialHours.value = Number(savedHours)
  }
  const savedType = localStorage.getItem(ENTRY_TYPE_KEY)
  if (isEntryType(savedType)) {
    entryType.value = savedType
  }
}

function selectType(type: EntryType) {
  entryType.value = type
  localStorage.setItem(ENTRY_TYPE_KEY, type)
}

function saveInitialHours() {
  if (initialHours.value != null && initialHours.value > 0) {
    localStorage.setItem(INITIAL_HOURS_KEY, String(initialHours.value))
  } else {
    initialHours.value = null
    localStorage.removeItem(INITIAL_HOURS_KEY)
  }
}

function save() {
  if (minutes.value == null) return
  entries.value.push({
    date: date.value,
    value: minutes.value,
    type: entryType.value,
    note: note.value || undefined,
  })
  persist()
  minutes.value = null
  note.value = ''
}

const sortedForTable = computed(() =>
  [...entries.value].sort((a, b) => b.date.localeCompare(a.date))
)

const hasOtherTypes = computed(() => entries.value.some(e => e.type !== 'input'))

// Sum minutes per type, grouped by the given key (date, week, month)
function totalsBy(keyOf: (e: TimeTrackerEntry) => string): Map<string, TypeTotals> {
  const map = new Map<string, TypeTotals>()
  for (const e of entries.value) {
    const key = keyOf(e)
    let totals = map.get(key)
    if (!totals) {
      totals = { input: 0, talking: 0, reading: 0 }
      map.set(key, totals)
    }
    totals[e.type] += e.value
  }
  return map
}

const dailyTotals = computed(() => totalsBy(e => e.date))

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
        minutes: dailyTotals.value.get(iso)?.input ?? 0,
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
  { min: 1, color: 'hsl(80,66%,80%)' },
  { min: 15, color: 'hsl(116,50%,78%)' },
  { min: 30, color: 'hsl(136,50%,66%)' },
  { min: 60, color: 'hsl(152,38%,56%)' },
  { min: 90, color: 'hsl(170,54%,37%)' },
  { min: 120, color: 'hsl(186,100%,24%)' },
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

function triggerImport() {
  importInput.value?.click()
}

function parseCSV(text: string): TimeTrackerEntry[] {
  const lines = text.trim().split('\n')
  if (lines.length < 2) throw new Error('Empty CSV')
  const result: TimeTrackerEntry[] = []
  for (let i = 1; i < lines.length; i++) {
    const line = lines[i]!
    // Match: date, minutes, optional type, quoted note (matching the export format)
    const m = line.match(
      /^(\d{4}-\d{2}-\d{2}),(\d+),(?:(input|talking|reading),)?("(?:[^"]|"")*"|.*)$/
    )
    if (!m) continue
    const date = m[1]!
    const value = Number(m[2])
    const type = (m[3] as EntryType | undefined) ?? 'input'
    let note = m[4]!
    if (note.startsWith('"') && note.endsWith('"')) {
      note = note.slice(1, -1).replace(/""/g, '"')
    }
    result.push({ date, value, type, note: note || undefined })
  }
  return result
}

function importFile(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = () => {
    try {
      const text = reader.result as string
      const data = file.name.endsWith('.csv')
        ? parseCSV(text)
        : (JSON.parse(text) as TimeTrackerEntry[])
      if (!Array.isArray(data)) throw new Error('Not an array')
      entries.value = withTypes(data)
      persist()
    } catch {
      alert('Invalid file.')
    }
  }
  reader.readAsText(file)
  // Reset so the same file can be re-imported
  ;(event.target as HTMLInputElement).value = ''
}

function clearAll() {
  if (!confirm('Delete all data? This cannot be undone.')) return
  entries.value = []
  initialHours.value = null
  localStorage.removeItem(STORAGE_KEY)
  localStorage.removeItem(INITIAL_HOURS_KEY)
}

function exportCSV() {
  const rows = [
    'date,minutes,type,note',
    ...entries.value.map(
      e => `${e.date},${e.value},${e.type},"${(e.note ?? '').replace(/"/g, '""')}"`
    ),
  ]
  const blob = new Blob([rows.join('\n')], { type: 'text/csv' })
  download(blob, 'timetracker.csv')
}

const importInput = ref<HTMLInputElement | null>(null)

const editingDate = ref<string | null>(null)
const editingValues = ref<Record<EntryType, number | null>>({
  input: null,
  talking: null,
  reading: null,
})
const editInputs = ref<HTMLInputElement[]>([])

function tapCell(cellDate: string) {
  const totals = dailyTotals.value.get(cellDate)
  editingDate.value = cellDate
  editingValues.value = {
    input: totals?.input || null,
    talking: totals?.talking || null,
    reading: totals?.reading || null,
  }
  nextTick(() => {
    const el = editInputs.value.find(i => i.dataset.type === entryType.value)
    el?.focus()
    el?.select()
  })
}

function commitEdit() {
  const cellDate = editingDate.value
  if (!cellDate) return
  const totals = dailyTotals.value.get(cellDate)

  for (const type of ENTRY_TYPES) {
    const val = Number(editingValues.value[type]) || 0
    // Only replace entries for types whose total actually changed
    if (val === (totals?.[type] ?? 0)) continue
    entries.value = entries.value.filter(e => e.date !== cellDate || e.type !== type)
    if (val > 0) {
      entries.value.push({ date: cellDate, value: val, type })
    }
  }
  persist()
  cancelEdit()
}

const totalHours = computed(() => {
  const totalMin = entries.value
    .filter(e => e.type === 'input')
    .reduce((sum, e) => sum + e.value, 0)
  return (totalMin / 60 + (initialHours.value ?? 0)).toFixed(1)
})

function formatHours(minutes: number): string {
  return (minutes / 60).toFixed(1) + 'h'
}

const STACKED_TYPES = ['talking', 'reading'] as const

function weekStartDate(dateStr: string): string {
  const d = new Date(dateStr + 'T00:00:00')
  d.setDate(d.getDate() - d.getDay()) // rewind to Sunday
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`
}

const weeklyChartData = computed(() => {
  const map = totalsBy(e => weekStartDate(e.date))
  if (map.size === 0)
    return {
      labels: [],
      weekStarts: [] as string[],
      weekly: { talking: [] as number[], reading: [] as number[] },
      datasets: [],
    }

  const weeks = [...map.keys()].sort()
  const firstWeek = weeks[0]!
  const lastWeek = weeks[weeks.length - 1]!

  // Fill in all weeks between first and last (including gaps with 0)
  const allWeeks: string[] = []
  const cursor = new Date(firstWeek + 'T00:00:00')
  const end = new Date(lastWeek + 'T00:00:00')
  while (cursor <= end) {
    allWeeks.push(
      `${cursor.getFullYear()}-${String(cursor.getMonth() + 1).padStart(2, '0')}-${String(cursor.getDate()).padStart(2, '0')}`
    )
    cursor.setDate(cursor.getDate() + 7)
  }

  const weeklyFor = (type: EntryType) => allWeeks.map(w => map.get(w)?.[type] ?? 0)
  const weeklyData = weeklyFor('input')
  let runningTotal = (initialHours.value ?? 0) * 60
  const priorCumulative: number[] = []
  for (const v of weeklyData) {
    priorCumulative.push(runningTotal)
    runningTotal += v
  }

  // Talking and reading are shown as running totals through each week
  const weekly = { talking: weeklyFor('talking'), reading: weeklyFor('reading') }
  const cumulativeFor = (data: number[]) => {
    let total = 0
    return data.map(v => (total += v))
  }
  // Build x-axis labels: show month name on the first week of each month
  const labelledMonths = new Set<string>()
  const xLabels = allWeeks.map(w => {
    const sun = new Date(w + 'T00:00:00')
    // Check if any day in this week (Sun–Sat) is the 1st of a month
    for (let d = 0; d < 7; d++) {
      const day = new Date(sun)
      day.setDate(day.getDate() + d)
      if (day.getDate() === 1) {
        const key = `${day.getFullYear()}-${day.getMonth()}`
        if (!labelledMonths.has(key)) {
          labelledMonths.add(key)
          if (day.getMonth() === 0) {
            return day.toLocaleString('default', { month: 'short' }) + ' ' + day.getFullYear()
          }
          return day.toLocaleString('default', { month: 'short' })
        }
      }
    }
    return ''
  })

  return {
    labels: xLabels,
    weekStarts: allWeeks,
    weekly,
    datasets: [
      {
        label: 'Cumulative',
        data: priorCumulative,
        backgroundColor: INPUT_COLORS.cumulative,
        stack: 'stack',
      },
      {
        label: 'This Week',
        data: weeklyData,
        backgroundColor: INPUT_COLORS.thisWeek,
        stack: 'stack',
      },
      // Talking/reading get their own stack, overlapping the input stack
      // (grouped: false) and drawn in front of it (lower order draws on top)
      ...STACKED_TYPES.map(type => ({
        label: type === 'talking' ? 'Talking' : 'Reading',
        data: cumulativeFor(weekly[type]),
        backgroundColor: TYPE_COLORS[type],
        stack: 'other',
        grouped: false,
        order: -1,
      })),
    ],
  }
})

const monthlyChartData = computed(() => {
  const map = totalsBy(e => e.date.slice(0, 7)) // "YYYY-MM"
  if (map.size === 0) return { labels: [], months: [] as string[], datasets: [] }

  const months = [...map.keys()].sort()
  const first = months[0]!
  const last = months[months.length - 1]!

  // Fill in all months between first and last (including gaps with 0)
  const allMonths: string[] = []
  const [startY, startM] = first.split('-').map(Number) as [number, number]
  const [endY, endM] = last.split('-').map(Number) as [number, number]
  let curY = startY,
    curM = startM
  while (curY < endY || (curY === endY && curM <= endM)) {
    allMonths.push(`${curY}-${String(curM).padStart(2, '0')}`)
    curM++
    if (curM > 12) {
      curM = 1
      curY++
    }
  }

  const monthlyFor = (type: EntryType) => allMonths.map(m => map.get(m)?.[type] ?? 0)
  const labels = allMonths.map(m => {
    const [y, mo] = m.split('-').map(Number) as [number, number]
    const d = new Date(y, mo - 1)
    if (mo === 1) return d.toLocaleString('default', { month: 'short' }) + ' ' + y
    return d.toLocaleString('default', { month: 'short' })
  })

  return {
    labels,
    months: allMonths,
    datasets: [
      {
        label: 'Input',
        data: monthlyFor('input'),
        backgroundColor: INPUT_COLORS.cumulative,
        stack: 'stack',
      },
      ...STACKED_TYPES.map(type => ({
        label: type === 'talking' ? 'Talking' : 'Reading',
        data: monthlyFor(type),
        backgroundColor: TYPE_COLORS[type],
        stack: 'stack',
      })),
    ],
  }
})

const monthlyChartOptions = computed(() => ({
  responsive: true,
  maintainAspectRatio: false,
  barPercentage: 0.95,
  categoryPercentage: 1.0,
  plugins: {
    legend: { display: false },
    tooltip: {
      mode: 'index' as const,
      intersect: false,
      callbacks: {
        title: (items: { dataIndex: number }[]) => {
          const idx = items[0]?.dataIndex
          if (idx == null) return ''
          const m = monthlyChartData.value.months[idx]
          if (!m) return ''
          const [y, mo] = m.split('-').map(Number) as [number, number]
          return new Date(y, mo - 1).toLocaleString('default', { month: 'long', year: 'numeric' })
        },
        label: (ctx: { dataset: { label?: string }; parsed: { y: number | null } }) => {
          const minutes = ctx.parsed.y ?? 0
          // Returning '' omits the line from the tooltip
          if (minutes <= 0) return ''
          return `${ctx.dataset.label}: ${(minutes / 60).toFixed(1)} hours`
        },
      },
    },
  },
  scales: {
    x: { stacked: true, ticks: { color: '#888', font: { size: 10 } }, grid: { display: false } },
    y: {
      stacked: true,
      ticks: {
        color: '#888',
        stepSize: 300,
        callback: (v: number | string) => Math.round(Number(v) / 60) + 'h',
      },
      grid: { color: 'rgba(255,255,255,0.06)' },
    },
  },
}))

const weeklyChartOptions = computed(() => ({
  responsive: true,
  maintainAspectRatio: false,
  barPercentage: 0.98,
  categoryPercentage: 1.0,
  plugins: {
    legend: { display: false },
    tooltip: {
      mode: 'index' as const,
      intersect: false,
      // Keep input lines first even though talking/reading are drawn on top
      itemSort: (a: { datasetIndex: number }, b: { datasetIndex: number }) =>
        a.datasetIndex - b.datasetIndex,
      callbacks: {
        title: (items: { dataIndex: number }[]) => {
          if (items.length === 0) return ''
          const idx = items[0]?.dataIndex
          if (idx == null) return ''
          const dateStr = weeklyChartData.value.weekStarts[idx]
          if (!dateStr) return ''
          const d = new Date(dateStr + 'T00:00:00')
          return (
            'Week starting ' +
            d.toLocaleDateString('default', { month: 'long', day: '2-digit', year: 'numeric' })
          )
        },
        label: (ctx: {
          datasetIndex: number
          dataIndex: number
          dataset: { label?: string }
          parsed: { y: number | null }
        }) => {
          const minutes = ctx.parsed.y ?? 0
          // Returning '' omits the line from the tooltip
          if (minutes <= 0) return ''
          let label = `${ctx.dataset.label}: ${formatHours(minutes)}`
          // Talking/reading bars are running totals; also show this week's amount
          const type = STACKED_TYPES[ctx.datasetIndex - 2]
          if (type) {
            const thisWeek = weeklyChartData.value.weekly[type][ctx.dataIndex] ?? 0
            if (thisWeek > 0) label += ` (+${formatHours(thisWeek)} this week)`
          }
          return label
        },
      },
    },
  },
  scales: {
    x: {
      stacked: true,
      ticks: { color: '#888', font: { size: 10 }, autoSkip: false, maxRotation: 90 },
      grid: { display: false },
    },
    y: {
      stacked: true,
      min: 0,
      max: (() => {
        const ds = weeklyChartData.value.datasets as { data: number[]; stack: string }[]
        const len = ds[0]?.data.length ?? 0
        let maxStack = 0
        for (let i = 0; i < len; i++) {
          const sums = new Map<string, number>()
          for (const d of ds) sums.set(d.stack, (sums.get(d.stack) ?? 0) + (d.data[i] ?? 0))
          maxStack = Math.max(maxStack, ...sums.values())
        }
        return Math.floor(maxStack / 3000) * 3000 + 3000
      })(),
      ticks: {
        color: '#888',
        stepSize: 3000,
        callback: (v: number | string) => Math.round(Number(v) / 60) + 'h',
      },
      grid: { color: 'rgba(255,255,255,0.06)' },
    },
  },
}))

function cancelEdit() {
  editingDate.value = null
  editingValues.value = { input: null, talking: null, reading: null }
}

function removeEntry(entry: TimeTrackerEntry) {
  const idx = entries.value.indexOf(entry)
  if (idx !== -1) {
    entries.value.splice(idx, 1)
    persist()
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
  <h1>CI Time Tracker</h1>
  <div class="heatmap-title" style="text-transform: unset; text-align: center; margin-bottom: 1rem">
    <span style="line-height: 1rem; font-size: 2rem; font-weight: 900; color: #83d583">{{
      totalHours
    }}</span>
    hours of input
  </div>
  <form class="entry-form" @submit.prevent="save">
    <div class="entry-row">
      <input type="date" v-model="date" required />
      <input type="number" v-model.number="minutes" step="1" placeholder="minutes" required />
      <button type="submit">+</button>
    </div>
    <div class="type-row" role="group" aria-label="Entry type">
      <button
        v-for="t in ENTRY_TYPES"
        :key="t"
        type="button"
        class="type-btn"
        :class="[t, { active: entryType === t }]"
        :style="{ '--type-color': BUTTON_COLORS[t] }"
        :aria-pressed="entryType === t"
        @click="selectType(t)"
      >
        {{ t }}
      </button>
    </div>
    <input type="text" v-model="note" placeholder="notes" class="note-input" />
  </form>

  <div v-if="entries.length" class="chart-container">
    <div class="heatmap-title">
      Weekly progress
      <span v-if="hasOtherTypes" class="chart-legend">
        <!-- Input is drawn in two colors: running total below, this week on top -->
        <span
          ><i
            :style="{
              background: `linear-gradient(to top, ${INPUT_COLORS.cumulative} 50%, ${INPUT_COLORS.thisWeek} 50%)`,
            }"
          ></i
          >input</span
        >
        <span v-for="t in STACKED_TYPES" :key="t"
          ><i :style="{ backgroundColor: TYPE_COLORS[t] }"></i>{{ t }}</span
        >
      </span>
    </div>
    <div class="chart-wrap">
      <Bar :data="weeklyChartData" :options="weeklyChartOptions" />
    </div>
  </div>

  <div v-if="entries.length" class="chart-container">
    <div class="heatmap-title">
      Monthly totals
      <span v-if="hasOtherTypes" class="chart-legend">
        <span><i :style="{ backgroundColor: INPUT_COLORS.cumulative }"></i>input</span>
        <span v-for="t in STACKED_TYPES" :key="t"
          ><i :style="{ backgroundColor: TYPE_COLORS[t] }"></i>{{ t }}</span
        >
      </span>
    </div>
    <div class="chart-wrap">
      <Bar :data="monthlyChartData" :options="monthlyChartOptions" />
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
            @click="tapCell(cell.date)"
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
      <label v-for="t in ENTRY_TYPES" :key="t" class="modal-field">
        <span class="modal-label">{{ t }}</span>
        <input
          ref="editInputs"
          :data-type="t"
          type="number"
          inputmode="numeric"
          class="modal-input"
          v-model.number="editingValues[t]"
          placeholder="Minutes"
          @keydown.enter="commitEdit"
          @keydown.escape="cancelEdit"
        />
      </label>
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
        <th v-for="t in ENTRY_TYPES" :key="t">{{ t }}</th>
        <th>Note</th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      <tr
        v-for="(entry, i) in sortedForTable"
        :key="`${entry.date}-${entry.type}-${i}`"
        class="entry-row-tap"
        @click="tapCell(entry.date)"
      >
        <td>{{ entry.date }}</td>
        <td v-for="t in ENTRY_TYPES" :key="t">{{ entry.type === t ? entry.value : '' }}</td>
        <td class="note-cell">{{ entry.note ?? '' }}</td>
        <td class="delete-cell">
          <span class="delete-btn" @click.stop="removeEntry(entry)">&times;</span>
        </td>
      </tr>
    </tbody>
  </table>

  <div class="initial-hours">
    <div class="initial-hours-label">Set initial hours of input</div>
    <div class="initial-hours-hint">
      This will be added to the running total of the entries above
    </div>
    <input
      type="number"
      v-model.number="initialHours"
      step="any"
      placeholder="0"
      class="initial-hours-input"
      @change="saveInitialHours"
    />
  </div>

  <div class="initial-hours">
    <div class="initial-hours-label">Data management</div>
    <p v-if="entries.length" class="export">
      Export data as:&nbsp;
      <a href="#" @click.prevent="exportCSV">CSV</a>&nbsp;
      <a href="#" @click.prevent="exportJSON">JSON</a>
    </p>

    <p class="export">
      <a href="#" @click.prevent="triggerImport">Import data</a>
      <input ref="importInput" type="file" accept=".json,.csv" hidden @change="importFile" />
    </p>

    <p v-if="entries.length" class="export">
      <a href="#" @click.prevent="clearAll">Clear all data</a>
    </p>
    <br />
    <hr />
    <p style="font-size: 0.75rem; color: #888">
      This site is open source with no trackers, cookies, etc.<br />Code is on
      <a href="https://github.com/billyc/timetracker">GitHub</a>.
    </p>
  </div>
</template>

<style scoped>
h1 {
  margin: 0.5rem 0 0.75rem 0;
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

.type-row {
  display: flex;
  gap: 0.5rem;
}

.type-btn {
  flex: 1;
  padding: 0.4em 0.5em;
  font-size: 0.85em;
  border: 1px solid #444;
  color: #aaa;
}

.type-btn.active {
  background: var(--type-color);
  border-color: var(--type-color);
  color: #fff;
}

/* White text is hard to read on yellow */
.type-btn.active.talking {
  color: #213547;
}

.chart-legend {
  float: right;
  font-size: 0.75em;
  font-weight: 500;
  text-transform: none;
  color: #888;
}

.chart-legend span {
  margin-left: 0.6em;
}

.chart-legend i {
  display: inline-block;
  width: 0.7em;
  height: 0.7em;
  margin-right: 0.25em;
  border-radius: 2px;
}

.note-input {
  width: 100%;
  box-sizing: border-box;
  display: none;
}

.heatmap-container {
  max-width: 400px;
  margin: 0 auto 1rem;
}

.heatmap-title {
  font-weight: 800;
  margin-top: 0.5rem;
  margin-bottom: 0rem;
  text-align: left;
  text-transform: uppercase;
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

.modal-field {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.5rem;
}

.modal-label {
  width: 4.5rem;
  text-align: left;
  font-size: 0.9em;
}

.modal-input {
  flex: 1;
  min-width: 0;
  box-sizing: border-box;
  padding: 0.5em 0.75em;
  border-radius: 8px;
  border: 1px solid #444;
  background: #1a1a1a;
  color: inherit;
  font-size: 1.1em;
  text-align: center;
}

.modal-buttons {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.75rem;
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

.entry-row-tap {
  cursor: pointer;
}

.note-cell {
  text-align: left;
  word-break: break-word;
}

.export {
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

.initial-hours {
  max-width: 400px;
  margin: 1.5rem auto 0;
  text-align: left;
}

.initial-hours-label {
  font-weight: 600;
  font-size: 0.9em;
}

.initial-hours-input {
  width: 6rem;
  margin-top: 0.35rem;
  padding: 0.5em 0.75em;
  border-radius: 8px;
  border: 1px solid #444;
  background: #1a1a1a;
  color: inherit;
  font-size: 1em;
}

.initial-hours-hint {
  margin-top: 0.25rem;
  font-size: 0.75em;
  color: #888;
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
  .type-btn {
    border-color: #ccc;
    color: #666;
  }
  .modal-input,
  .initial-hours-input {
    background: #f9f9f9;
    border-color: #ccc;
  }
}
</style>
