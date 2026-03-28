<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import { apiFetch } from '../api'

type Session = {
  id: number
  operation_id: number
  started_at: string
  stopped_at: string
  duration_seconds: number
  work_date: string
  operation_name: string
}

function todayLocal(): string {
  const d = new Date()
  const y = d.getFullYear()
  const m = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${y}-${m}-${day}`
}

const date = ref(todayLocal())
const sessions = ref<Session[]>([])
const err = ref('')
const busy = ref(false)

const totalSeconds = computed(() => sessions.value.reduce((a, s) => a + s.duration_seconds, 0))

type SessionGroup = {
  operation_name: string
  total_seconds: number
  sessions: Session[]
}

const groupedSessions = computed((): SessionGroup[] => {
  const map = new Map<string, Session[]>()
  for (const s of sessions.value) {
    const key = s.operation_name
    if (!map.has(key)) map.set(key, [])
    map.get(key)!.push(s)
  }
  const groups: SessionGroup[] = []
  for (const [operation_name, sess] of map) {
    sess.sort((a, b) => a.started_at.localeCompare(b.started_at))
    groups.push({
      operation_name,
      total_seconds: sess.reduce((a, x) => a + x.duration_seconds, 0),
      sessions: sess,
    })
  }
  groups.sort((a, b) => a.operation_name.localeCompare(b.operation_name, 'bg'))
  return groups
})

function formatHms(total: number): string {
  const h = Math.floor(total / 3600)
  const m = Math.floor((total % 3600) / 60)
  const s = total % 60
  return [h, m, s].map((n) => String(n).padStart(2, '0')).join(':')
}

async function load() {
  err.value = ''
  busy.value = true
  try {
    const q = new URLSearchParams({ date: date.value })
    const r = await apiFetch<{ sessions: Session[] }>(`sessions?${q.toString()}`)
    sessions.value = r.sessions
  } catch (e) {
    err.value = e instanceof Error ? e.message : 'Грешка'
  } finally {
    busy.value = false
  }
}

onMounted(load)
watch(date, load)

function exportText(): void {
  const blocks: string[] = []
  for (const g of groupedSessions.value) {
    blocks.push(`${g.operation_name}\t(общо)\t\t${formatHms(g.total_seconds)}`)
    for (const s of g.sessions) {
      blocks.push(`\t${s.started_at}\t${s.stopped_at}\t${formatHms(s.duration_seconds)}`)
    }
    blocks.push('')
  }
  if (blocks.length && blocks[blocks.length - 1] === '') blocks.pop()
  const lines = [
    `Работен график — ${date.value}`,
    `Общо време: ${formatHms(totalSeconds.value)}`,
    '',
    ...blocks,
  ]
  const blob = new Blob([lines.join('\n')], { type: 'text/plain;charset=utf-8' })
  const a = document.createElement('a')
  a.href = URL.createObjectURL(blob)
  a.download = `grafik-${date.value}.txt`
  a.click()
  URL.revokeObjectURL(a.href)
}

async function copyText(): Promise<void> {
  const blocks: string[] = []
  for (const g of groupedSessions.value) {
    blocks.push(`${g.operation_name} | Общо: ${formatHms(g.total_seconds)}`)
    for (const s of g.sessions) {
      blocks.push(
        `  ${s.started_at} – ${s.stopped_at} | ${formatHms(s.duration_seconds)}`
      )
    }
    blocks.push('')
  }
  if (blocks.length && blocks[blocks.length - 1] === '') blocks.pop()
  const body = [
    `Работен график — ${date.value}`,
    `Общо: ${formatHms(totalSeconds.value)}`,
    '',
    ...blocks,
  ].join('\n')
  try {
    await navigator.clipboard.writeText(body)
  } catch {
    err.value = 'Копирането не е възможно на това устройство'
  }
}
</script>

<template>
  <div class="card">
    <label class="muted" for="d">Дата</label>
    <input id="d" v-model="date" type="date" class="input" style="margin-top: 0.35rem; margin-bottom: 1rem" />

    <p class="muted" style="margin: 0 0 0.5rem">Общо за деня: <strong>{{ formatHms(totalSeconds) }}</strong></p>

    <div
      v-for="g in groupedSessions"
      :key="g.operation_name"
      class="list-row"
      style="flex-direction: column; align-items: stretch"
    >
      <div
        style="display: flex; justify-content: space-between; align-items: baseline; gap: 0.5rem; flex-wrap: wrap"
      >
        <span style="font-weight: 600">{{ g.operation_name }}</span>
        <span class="muted" style="font-size: 0.9rem">Общо: {{ formatHms(g.total_seconds) }}</span>
      </div>
      <ul class="muted" style="font-size: 0.85rem; margin: 0.4rem 0 0; padding-left: 1.1rem; list-style: disc">
        <li v-for="s in g.sessions" :key="s.id">
          {{ s.started_at?.slice(11, 19) }} – {{ s.stopped_at?.slice(11, 19) }} · {{ formatHms(s.duration_seconds) }}
        </li>
      </ul>
    </div>
    <p v-if="!busy && sessions.length === 0" class="muted">Няма записи за тази дата.</p>
    <p v-if="err" class="err">{{ err }}</p>

    <div style="display: flex; flex-direction: column; gap: 0.5rem; margin-top: 1rem">
      <button class="btn btn-ghost" type="button" :disabled="sessions.length === 0" @click="exportText">
        Експорт .txt
      </button>
      <button class="btn btn-ghost" type="button" :disabled="sessions.length === 0" @click="copyText">
        Копирай текст
      </button>
    </div>
  </div>
</template>
