<script setup lang="ts">
import { ref, computed, onMounted, defineComponent, h } from 'vue'

// =========================================================================
// types
// =========================================================================
interface Exercise {
  id: number
  position: number
  slot: number
  name: string
  target_sets: number | null
  target_reps: number | null
  notes: string | null
}
interface Group { position: number; exercises: Exercise[] }
interface MenuSummary {
  id: number
  name: string
  description: string | null
  created_by: number
  created_by_username: string
  created_at: string
  exercise_count: number
}
interface MenuDetail { menu: MenuSummary; groups: Group[] }

interface SetEntry {
  exerciseId: number
  exerciseName: string
  weight: string   // 用 string 方便 input 綁定
  reps: string
}
interface LogPerExercise {
  exercise: Exercise
  sets: SetEntry[]
}

interface StatRow {
  exercise_name: string
  user_id: number
  username: string
  display_name: string | null
  day: string
  max_weight: string | null
  total_volume: string | null
  set_count: number
}
interface MyLogRow {
  id: number
  performed_at: string
  menu_id: number | null
  menu_name: string | null
  notes: string | null
  set_count: number
}

// 取代 useAuth (我們沒在這個 sub-project 裡能直接用 portal-main 的 composables)
interface Me { id: number; username: string; role: string }

// =========================================================================
// state
// =========================================================================
const me = ref<Me | null>(null)
const tab = ref<'menus' | 'log' | 'stats'>('menus')

const menus = ref<MenuSummary[]>([])
const menuDetail = ref<MenuDetail | null>(null)
const loading = ref(false)
const errorMsg = ref('')
const okMsg = ref('')

// 建菜單草稿 (admin)
interface DraftExercise {
  name: string
  position: number
  slot: number
  target_sets: string
  target_reps: string
  notes: string
}
interface Draft {
  name: string
  description: string
  exercises: DraftExercise[]
}
const showBuilder = ref(false)
const draft = ref<Draft>({
  name: '',
  description: '',
  exercises: [{ name: '', position: 1, slot: 1, target_sets: '', target_reps: '', notes: '' }],
})

// 紀錄表單
const logMenuId = ref<number | null>(null)
const logRows = ref<LogPerExercise[]>([])
const logNotes = ref('')

// 統計
const stats = ref<StatRow[]>([])
const myLogs = ref<MyLogRow[]>([])

// =========================================================================
// helpers
// =========================================================================
async function fetchMe() {
  try {
    const r = await fetch('/api/auth/me', { credentials: 'include' })
    const d = await r.json()
    if (d?.user) me.value = { id: d.user.id, username: d.user.username, role: d.user.role }
  } catch {}
}

const isAdmin = computed(() => me.value?.role === 'admin')

async function loadMenus() {
  loading.value = true
  errorMsg.value = ''
  try {
    const r = await fetch('/api/fitness/menus', { credentials: 'include' })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    const d = await r.json()
    menus.value = d.menus
  } catch (e: any) { errorMsg.value = e?.message || '載入失敗' }
  finally { loading.value = false }
}

async function openMenu(id: number) {
  errorMsg.value = ''
  loading.value = true
  try {
    const r = await fetch(`/api/fitness/menus/${id}`, { credentials: 'include' })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    menuDetail.value = await r.json()
  } catch (e: any) { errorMsg.value = e?.message || '載入失敗' }
  finally { loading.value = false }
}

async function deleteMenu(id: number) {
  if (!confirm('確定刪除這個菜單？歷史紀錄保留但 menu_id 會清空。')) return
  try {
    const r = await fetch(`/api/admin/fitness/menus/${id}`, { method: 'DELETE', credentials: 'include' })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    menus.value = menus.value.filter(m => m.id !== id)
    if (menuDetail.value?.menu.id === id) menuDetail.value = null
  } catch (e: any) { errorMsg.value = e?.message || '刪除失敗' }
}

// ---- builder ----
function addSlot(position: number) {
  const lastSlot = Math.max(...draft.value.exercises.filter(e => e.position === position).map(e => e.slot), 0)
  draft.value.exercises.push({
    name: '', position, slot: lastSlot + 1, target_sets: '', target_reps: '', notes: '',
  })
}
function addPosition() {
  const lastPos = Math.max(...draft.value.exercises.map(e => e.position), 0)
  draft.value.exercises.push({
    name: '', position: lastPos + 1, slot: 1, target_sets: '', target_reps: '', notes: '',
  })
}
function removeExercise(idx: number) {
  if (draft.value.exercises.length <= 1) return
  draft.value.exercises.splice(idx, 1)
}
function resetDraft() {
  draft.value = {
    name: '', description: '',
    exercises: [{ name: '', position: 1, slot: 1, target_sets: '', target_reps: '', notes: '' }],
  }
}
async function saveMenu() {
  errorMsg.value = ''
  okMsg.value = ''
  if (!draft.value.name.trim()) { errorMsg.value = '菜單名稱必填'; return }
  const exs = draft.value.exercises
    .filter(e => e.name.trim())
    .map(e => ({
      name: e.name.trim(),
      position: e.position,
      slot: e.slot,
      target_sets: e.target_sets ? Number(e.target_sets) : null,
      target_reps: e.target_reps ? Number(e.target_reps) : null,
      notes: e.notes || null,
    }))
  if (!exs.length) { errorMsg.value = '至少要一個動作'; return }
  try {
    const r = await fetch('/api/admin/fitness/menus', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({
        name: draft.value.name.trim(),
        description: draft.value.description.trim() || null,
        exercises: exs,
      }),
    })
    if (!r.ok) {
      const err = await r.json().catch(() => ({}))
      throw new Error(err?.statusMessage || `HTTP ${r.status}`)
    }
    okMsg.value = '✓ 菜單已建立'
    showBuilder.value = false
    resetDraft()
    await loadMenus()
  } catch (e: any) { errorMsg.value = e?.message || '儲存失敗' }
}

// ---- 紀錄 ----
const exerciseGroups = computed(() => menuDetail.value?.groups || [])
const flatExercises = computed(() => exerciseGroups.value.flatMap(g => g.exercises))

async function startLog(menuId: number) {
  await openMenu(menuId)
  if (!menuDetail.value) return
  logMenuId.value = menuId
  // 為每個 exercise 預先建好 set 列 (用 target_sets 或 1)
  logRows.value = flatExercises.value.map(ex => {
    const numSets = ex.target_sets || 1
    const sets: SetEntry[] = []
    for (let i = 0; i < numSets; i++) {
      sets.push({
        exerciseId: ex.id, exerciseName: ex.name,
        weight: '', reps: ex.target_reps?.toString() || '',
      })
    }
    return { exercise: ex, sets }
  })
  logNotes.value = ''
  tab.value = 'log'
}

function addSetRow(exerciseIdx: number) {
  const row = logRows.value[exerciseIdx]
  if (!row) return
  const last = row.sets[row.sets.length - 1]
  row.sets.push({
    exerciseId: row.exercise.id, exerciseName: row.exercise.name,
    weight: last?.weight || '', reps: last?.reps || '',
  })
}
function removeSetRow(exerciseIdx: number, setIdx: number) {
  const row = logRows.value[exerciseIdx]
  if (!row || row.sets.length <= 1) return
  row.sets.splice(setIdx, 1)
}

async function saveLog() {
  errorMsg.value = ''
  okMsg.value = ''
  const sets: any[] = []
  for (const row of logRows.value) {
    row.sets.forEach((s, i) => {
      // 至少要有 weight 或 reps 才存
      if (!s.weight && !s.reps) return
      sets.push({
        exerciseId: row.exercise.id,
        exerciseName: row.exercise.name,
        setNumber: i + 1,
        weightKg: s.weight ? Number(s.weight) : null,
        reps: s.reps ? Number(s.reps) : null,
      })
    })
  }
  if (!sets.length) { errorMsg.value = '至少要填一組重量或次數'; return }
  try {
    const r = await fetch('/api/fitness/logs', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({
        menuId: logMenuId.value,
        notes: logNotes.value.trim() || null,
        sets,
      }),
    })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    okMsg.value = '✓ 紀錄已儲存'
    // 清空，留在這頁讓使用者繼續 (or 切到 stats)
    logRows.value = []
    logMenuId.value = null
    menuDetail.value = null
    await loadStats()
    tab.value = 'stats'
  } catch (e: any) { errorMsg.value = e?.message || '儲存失敗' }
}

// ---- stats ----
async function loadStats() {
  loading.value = true
  errorMsg.value = ''
  try {
    const r = await fetch('/api/fitness/stats', { credentials: 'include' })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    const d = await r.json()
    stats.value = d.stats
    myLogs.value = d.myLogs
  } catch (e: any) { errorMsg.value = e?.message || '載入失敗' }
  finally { loading.value = false }
}

// 把 stats 按 exercise 分組，每個 exercise 內按 user 分組
interface ChartUserSeries { userId: number; username: string; points: { day: string; value: number }[] }
interface ChartData { exerciseName: string; series: ChartUserSeries[] }
const chartsByExercise = computed<ChartData[]>(() => {
  const byEx = new Map<string, Map<number, ChartUserSeries>>()
  for (const r of stats.value) {
    const w = r.max_weight ? Number(r.max_weight) : null
    if (w === null || isNaN(w)) continue
    let users = byEx.get(r.exercise_name)
    if (!users) { users = new Map(); byEx.set(r.exercise_name, users) }
    let series = users.get(r.user_id)
    if (!series) {
      series = { userId: r.user_id, username: r.display_name || r.username, points: [] }
      users.set(r.user_id, series)
    }
    series.points.push({ day: r.day, value: w })
  }
  const out: ChartData[] = []
  for (const [name, users] of byEx) {
    out.push({ exerciseName: name, series: Array.from(users.values()) })
  }
  return out.sort((a, b) => a.exerciseName.localeCompare(b.exerciseName))
})

// 顏色給每個 user 一個固定色 (依 username hash)
function colorFor(username: string): string {
  let h = 0
  for (const c of username) h = (h * 31 + c.charCodeAt(0)) >>> 0
  // 使用較亮的 hue
  const hue = h % 360
  return `hsl(${hue}, 60%, 65%)`
}

function fmtTime(s: string) { return new Date(s).toLocaleString() }
function fmtDay(s: string) { return s.slice(5) }     // MM-DD

// =========================================================================
// SVG line chart 子 component (避免拆檔)
// =========================================================================
const Chart = defineComponent({
  props: { data: { type: Object as any, required: true } },
  setup(props: any) {
    const W = 720
    const H = 220
    const pad = { l: 40, r: 16, t: 12, b: 28 }

    const dims = computed(() => {
      const all = props.data.series.flatMap((s: any) => s.points)
      if (!all.length) return null
      const days = Array.from(new Set(all.map((p: any) => p.day))).sort() as string[]
      const dayIdx = new Map<string, number>(days.map((d, i) => [d, i]))
      const minV = Math.min(...all.map((p: any) => p.value))
      const maxV = Math.max(...all.map((p: any) => p.value))
      const yMin = Math.max(0, minV - 5)
      const yMax = maxV + 5
      const innerW = W - pad.l - pad.r
      const innerH = H - pad.t - pad.b
      const xOf = (day: string) => days.length === 1
        ? pad.l + innerW / 2
        : pad.l + ((dayIdx.get(day) || 0) / (days.length - 1)) * innerW
      const yOf = (v: number) => pad.t + innerH - ((v - yMin) / (yMax - yMin || 1)) * innerH
      return { days, dayIdx, yMin, yMax, innerW, innerH, xOf, yOf }
    })

    const lineColor = (username: string) => {
      let h = 0
      for (const c of username) h = (h * 31 + c.charCodeAt(0)) >>> 0
      return `hsl(${h % 360}, 60%, 65%)`
    }

    return () => {
      const d = dims.value
      if (!d) return h('div', { class: 'chart-empty' }, '無資料')

      const yTicks: any[] = []
      const ySteps = 4
      for (let i = 0; i <= ySteps; i++) {
        const v = d.yMin + (d.yMax - d.yMin) * (i / ySteps)
        const y = d.yOf(v)
        yTicks.push(h('g', { key: 'yt' + i }, [
          h('line', {
            x1: pad.l, x2: W - pad.r, y1: y, y2: y,
            stroke: 'rgba(255,255,255,.05)', 'stroke-dasharray': '2 4',
          }),
          h('text', {
            x: pad.l - 6, y: y + 4, 'text-anchor': 'end',
            fill: 'rgba(255,255,255,.4)', 'font-size': 10,
          }, v.toFixed(0)),
        ]))
      }

      const xTicks: any[] = []
      const showEvery = Math.max(1, Math.ceil(d.days.length / 6))
      d.days.forEach((day: string, i: number) => {
        if (i % showEvery !== 0 && i !== d.days.length - 1) return
        const x = d.xOf(day)
        xTicks.push(h('text', {
          x, y: H - pad.b + 14, 'text-anchor': 'middle',
          fill: 'rgba(255,255,255,.4)', 'font-size': 10,
        }, day.slice(5)))
      })

      const lines: any[] = []
      const points: any[] = []
      for (const s of props.data.series) {
        const sorted = [...s.points].sort((a: any, b: any) => a.day.localeCompare(b.day))
        const pathD = sorted.map((p: any, i: number) =>
          `${i === 0 ? 'M' : 'L'} ${d.xOf(p.day).toFixed(1)} ${d.yOf(p.value).toFixed(1)}`,
        ).join(' ')
        const c = lineColor(s.username)
        lines.push(h('path', {
          d: pathD, fill: 'none', stroke: c,
          'stroke-width': 2, 'stroke-linecap': 'round',
        }))
        for (const p of sorted) {
          points.push(h('circle', {
            cx: d.xOf(p.day), cy: d.yOf(p.value), r: 3,
            fill: c, stroke: '#0d0d0d', 'stroke-width': 1.5,
          }, [h('title', null, `${s.username} · ${p.day} · ${p.value} kg`)]))
        }
      }

      return h('svg', {
        width: W, height: H, class: 'chart-svg', viewBox: `0 0 ${W} ${H}`,
      }, [...yTicks, ...xTicks, ...lines, ...points])
    }
  },
})

// =========================================================================
// boot
// =========================================================================
onMounted(async () => {
  await fetchMe()
  await loadMenus()
})
</script>

<template>
  <div class="fit">
    <!-- 頂部 tab nav -->
    <nav class="tabs">
      <button :class="['tab', { active: tab === 'menus' }]" @click="tab = 'menus'">
        <span class="tab-glyph">▤</span> 菜單
      </button>
      <button :class="['tab', { active: tab === 'log' }]" @click="tab = 'log'">
        <span class="tab-glyph">✎</span> 紀錄
      </button>
      <button :class="['tab', { active: tab === 'stats' }]" @click="() => { loadStats(); tab = 'stats' }">
        <span class="tab-glyph">📈</span> 大家的成長
      </button>
    </nav>

    <p v-if="errorMsg" class="alert">{{ errorMsg }}</p>
    <p v-if="okMsg" class="ok">{{ okMsg }}</p>

    <!-- ==================== 菜單 tab ==================== -->
    <section v-if="tab === 'menus'" class="page">
      <header class="head">
        <div>
          <p class="kicker">MENUS</p>
          <h2>健身菜單</h2>
        </div>
        <button v-if="isAdmin" class="btn-primary" @click="showBuilder = true">+ 新增菜單</button>
      </header>

      <div v-if="loading && !menus.length" class="muted">LOADING…</div>
      <div v-else-if="!menus.length" class="empty">尚無菜單<span v-if="isAdmin">，點上方「+ 新增菜單」開始</span></div>

      <div v-else class="menu-list">
        <article v-for="m in menus" :key="m.id" class="menu-card">
          <div class="menu-head">
            <div>
              <h3>{{ m.name }}</h3>
              <p class="muted">{{ m.description || '—' }}</p>
              <p class="meta">建立者 {{ m.created_by_username }} · {{ m.exercise_count }} 個動作 · {{ fmtTime(m.created_at) }}</p>
            </div>
            <div class="menu-actions">
              <button class="btn-ghost" @click="openMenu(m.id)">查看</button>
              <button class="btn-primary" @click="startLog(m.id)">▶ 開始紀錄</button>
              <button v-if="isAdmin" class="btn-ghost danger small" @click="deleteMenu(m.id)">刪除</button>
            </div>
          </div>

          <div v-if="menuDetail?.menu.id === m.id" class="menu-detail">
            <div v-for="g in menuDetail.groups" :key="g.position" class="group">
              <p class="group-label">
                <span class="kicker">POSITION {{ g.position }}</span>
                <span v-if="g.exercises.length > 1" class="superset-badge">SUPERSET ×{{ g.exercises.length }}</span>
              </p>
              <ul class="ex-list">
                <li v-for="ex in g.exercises" :key="ex.id">
                  <span class="ex-name">{{ ex.name }}</span>
                  <span v-if="ex.target_sets || ex.target_reps" class="ex-target">
                    {{ ex.target_sets || '?' }} × {{ ex.target_reps || '?' }}
                  </span>
                  <span v-if="ex.notes" class="ex-notes">— {{ ex.notes }}</span>
                </li>
              </ul>
            </div>
          </div>
        </article>
      </div>
    </section>

    <!-- ==================== 紀錄 tab ==================== -->
    <section v-if="tab === 'log'" class="page">
      <header class="head">
        <div>
          <p class="kicker">LOG SESSION</p>
          <h2>記錄這次訓練</h2>
        </div>
      </header>

      <div v-if="!logRows.length" class="empty">
        請先到「菜單」頁選一份菜單，按「▶ 開始紀錄」進來。
      </div>

      <div v-else class="log-form">
        <p class="muted">{{ flatExercises.length }} 個動作，依序填入重量 (kg) 與次數。空白格不會儲存。</p>

        <div v-for="(row, idx) in logRows" :key="row.exercise.id" class="log-block">
          <div class="log-block-head">
            <span class="kicker">{{ row.exercise.position }}.{{ row.exercise.slot }}</span>
            <h4>{{ row.exercise.name }}</h4>
            <span v-if="row.exercise.target_sets || row.exercise.target_reps" class="hint">
              建議 {{ row.exercise.target_sets || '?' }} × {{ row.exercise.target_reps || '?' }}
            </span>
          </div>

          <div class="set-grid">
            <div class="set-grid-head">
              <span>SET</span>
              <span>重量 (kg)</span>
              <span>次數</span>
              <span></span>
            </div>
            <div v-for="(s, sIdx) in row.sets" :key="sIdx" class="set-row">
              <span class="set-num">#{{ sIdx + 1 }}</span>
              <input v-model="s.weight" type="number" step="0.5" placeholder="0" />
              <input v-model="s.reps" type="number" placeholder="0" />
              <button class="ico" @click="removeSetRow(idx, sIdx)" :disabled="row.sets.length <= 1">×</button>
            </div>
          </div>
          <button class="btn-ghost small" @click="addSetRow(idx)">+ 加一組</button>
        </div>

        <label class="field">
          <span>備註 (整次訓練)</span>
          <textarea v-model="logNotes" rows="2" placeholder="今天身體狀況、感受…" />
        </label>

        <button class="btn-primary lg" @click="saveLog">儲存紀錄</button>
      </div>
    </section>

    <!-- ==================== 大家的成長 tab ==================== -->
    <section v-if="tab === 'stats'" class="page">
      <header class="head">
        <div>
          <p class="kicker">DASHBOARD</p>
          <h2>大家的成長曲線 (依動作分)</h2>
        </div>
        <button class="btn-ghost" @click="loadStats">↻ 重新載入</button>
      </header>

      <div v-if="loading && !chartsByExercise.length" class="muted">LOADING…</div>
      <div v-else-if="!chartsByExercise.length" class="empty">還沒有任何紀錄。先去記一次！</div>

      <div v-else class="charts">
        <article v-for="c in chartsByExercise" :key="c.exerciseName" class="chart-card">
          <header class="chart-head">
            <h3>{{ c.exerciseName }}</h3>
            <p class="muted">y 軸 = 該日最大重量 (kg)，x 軸 = 日期</p>
          </header>
          <Chart :data="c" />
          <ul class="legend">
            <li v-for="s in c.series" :key="s.userId">
              <span class="legend-dot" :style="{ background: colorFor(s.username) }" />
              {{ s.username }} <em>{{ s.points.length }} 天</em>
            </li>
          </ul>
        </article>
      </div>

      <header class="head" style="margin-top: 32px">
        <div>
          <p class="kicker">MY RECENT</p>
          <h2>我最近的訓練</h2>
        </div>
      </header>
      <table v-if="myLogs.length" class="logs-table">
        <thead>
          <tr><th>時間</th><th>菜單</th><th>組數</th><th>備註</th></tr>
        </thead>
        <tbody>
          <tr v-for="l in myLogs" :key="l.id">
            <td>{{ fmtTime(l.performed_at) }}</td>
            <td>{{ l.menu_name || '—' }}</td>
            <td class="num">{{ l.set_count }}</td>
            <td class="muted-cell">{{ l.notes || '' }}</td>
          </tr>
        </tbody>
      </table>
      <p v-else class="muted">尚無紀錄</p>
    </section>

    <!-- ==================== Builder modal ==================== -->
    <div v-if="showBuilder" class="overlay" @click.self="showBuilder = false">
      <div class="builder">
        <header class="builder-head">
          <p class="kicker">NEW MENU</p>
          <button class="ico" @click="showBuilder = false">×</button>
        </header>

        <div class="builder-body">
          <label class="field">
            <span>菜單名稱 *</span>
            <input v-model="draft.name" type="text" placeholder="例：上肢日 A" />
          </label>
          <label class="field">
            <span>描述 (選填)</span>
            <input v-model="draft.description" type="text" placeholder="例：胸 + 三頭主練" />
          </label>

          <p class="kicker">EXERCISES</p>
          <p class="muted small">同一 position 多個 slot = 超組合 (superset)。第一個 position 想做組合，按下方「+ 加超組合到此 group」</p>

          <template v-for="pos in [...new Set(draft.exercises.map(e => e.position))].sort((a, b) => a - b)" :key="pos">
            <div class="builder-group">
              <p class="group-label">
                <span class="kicker">POSITION {{ pos }}</span>
                <span v-if="draft.exercises.filter(e => e.position === pos).length > 1" class="superset-badge">
                  SUPERSET ×{{ draft.exercises.filter(e => e.position === pos).length }}
                </span>
              </p>
              <div
                v-for="(ex, idx) in draft.exercises"
                v-if="ex.position === pos"
                :key="idx"
                class="builder-ex"
              >
                <span class="ex-tag">{{ pos }}.{{ ex.slot }}</span>
                <input v-model="ex.name" type="text" placeholder="動作名稱 *" class="grow" />
                <input v-model="ex.target_sets" type="number" placeholder="組" class="num-input" />
                <input v-model="ex.target_reps" type="number" placeholder="次" class="num-input" />
                <input v-model="ex.notes" type="text" placeholder="備註" class="grow" />
                <button class="ico" @click="removeExercise(idx)" :disabled="draft.exercises.length <= 1">×</button>
              </div>
              <button class="btn-ghost tiny" @click="addSlot(pos)">+ 加超組合到此 group</button>
            </div>
          </template>

          <button class="btn-ghost" @click="addPosition">+ 加下一個 position (新動作)</button>
        </div>

        <footer class="builder-foot">
          <button class="btn-ghost" @click="showBuilder = false">取消</button>
          <button class="btn-primary" @click="saveMenu">儲存菜單</button>
        </footer>
      </div>
    </div>
  </div>
</template>


<style scoped>
.fit { display: flex; flex-direction: column; gap: 18px; }

/* tabs */
.tabs { display: flex; gap: 6px; flex-wrap: wrap; }
.tab {
  display: inline-flex; align-items: center; gap: 8px;
  padding: 10px 16px;
  background: var(--bg-tile);
  color: var(--tx-2);
  border: 1px solid var(--line-2);
  border-radius: var(--r-sm);
  font-size: 12.5px;
  letter-spacing: .04em;
  font-weight: 500;
  transition: background .15s, color .15s, border-color .15s;
}
.tab:hover { background: var(--hover-bg); color: var(--tx-1); }
.tab.active { background: var(--tx-1); color: #000; border-color: var(--tx-1); }
.tab-glyph { font-size: 14px; }

/* alerts */
.ok {
  background: rgba(0, 209, 138, .10);
  border: 1px solid rgba(0, 209, 138, .30);
  color: var(--success);
  font-size: 13px;
  padding: 10px 12px;
  border-radius: var(--r-sm);
}

/* page */
.page { display: flex; flex-direction: column; gap: 16px; }
.head { display: flex; justify-content: space-between; align-items: flex-end; }
.kicker { font-size: 10.5px; letter-spacing: .26em; color: var(--tx-3); margin: 0; font-weight: 500; }
.head h2 { font-size: 24px; font-weight: 500; margin-top: 6px; letter-spacing: -0.01em; }

.muted { color: var(--tx-2); font-size: 13.5px; }
.muted.small { font-size: 12px; }
.empty {
  text-align: center; padding: 60px 24px;
  background: var(--bg-tile);
  border: 1px dashed var(--line-2);
  border-radius: var(--r-md);
  color: var(--tx-3);
}

/* menu list */
.menu-list { display: flex; flex-direction: column; gap: 12px; }
.menu-card {
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
  padding: 18px 22px;
}
.menu-head {
  display: flex; justify-content: space-between; gap: 16px;
  flex-wrap: wrap;
}
.menu-head h3 { font-size: 17px; font-weight: 500; margin-bottom: 4px; }
.menu-head .meta { font-size: 11.5px; color: var(--tx-3); margin-top: 8px; letter-spacing: .04em; }
.menu-actions { display: flex; gap: 8px; flex-wrap: wrap; align-items: flex-start; }

.menu-detail {
  margin-top: 18px;
  padding-top: 16px;
  border-top: 1px solid var(--line-2);
  display: flex; flex-direction: column; gap: 14px;
}

.group-label {
  display: flex; align-items: center; gap: 10px;
  margin: 0 0 8px;
}
.superset-badge {
  font-size: 10px; letter-spacing: .14em; font-weight: 500;
  color: #fbbf24;
  background: rgba(245, 158, 11, .10);
  border: 1px solid rgba(245, 158, 11, .30);
  padding: 2px 8px; border-radius: 999px;
}
.ex-list { list-style: none; padding: 0; margin: 0; display: flex; flex-direction: column; gap: 6px; }
.ex-list li {
  display: flex; gap: 10px; align-items: baseline; flex-wrap: wrap;
  font-size: 13.5px;
}
.ex-name { color: var(--tx-1); font-weight: 500; }
.ex-target {
  color: var(--tx-2);
  background: var(--bg-input);
  border: 1px solid var(--line-2);
  padding: 1px 8px; border-radius: 999px;
  font-size: 11.5px;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
}
.ex-notes { color: var(--tx-3); font-size: 12.5px; }

/* log */
.log-form { display: flex; flex-direction: column; gap: 18px; }
.log-block {
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
  padding: 18px 20px;
}
.log-block-head {
  display: flex; align-items: baseline; gap: 12px; margin-bottom: 12px;
}
.log-block-head h4 { font-size: 16px; font-weight: 500; flex: 1; }
.log-block-head .hint { font-size: 12px; color: var(--tx-3); }

.set-grid { display: flex; flex-direction: column; gap: 6px; margin-bottom: 8px; }
.set-grid-head {
  display: grid;
  grid-template-columns: 50px 1fr 1fr 30px;
  gap: 10px;
  font-size: 10.5px; letter-spacing: .14em; color: var(--tx-3);
  padding-left: 4px;
}
.set-row {
  display: grid;
  grid-template-columns: 50px 1fr 1fr 30px;
  gap: 10px;
  align-items: center;
}
.set-num {
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  font-size: 12px; color: var(--tx-2);
}
.set-row input {
  width: 100%;
  padding: 8px 10px;
  border: 1px solid var(--line-2);
  background: var(--bg-input);
  color: var(--tx-1);
  border-radius: var(--r-sm);
  outline: none;
  font-size: 13px;
}
.set-row input:focus { border-color: var(--tx-2); }

.field span {
  display: block; font-size: 10.5px; color: var(--tx-3);
  letter-spacing: .14em; margin-bottom: 6px;
}
.field textarea {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid var(--line-2);
  background: var(--bg-input);
  color: var(--tx-1);
  border-radius: var(--r-sm);
  outline: none;
  font: inherit;
  font-size: 13px;
  resize: vertical;
}
.field textarea:focus { border-color: var(--tx-2); }

.btn-primary.lg { padding: 13px 24px; font-size: 13.5px; align-self: flex-start; }
.btn-ghost.small { padding: 4px 10px; font-size: 11.5px; }
.btn-ghost.tiny { padding: 4px 8px; font-size: 10.5px; }
.btn-ghost.danger:hover { color: var(--danger); border-color: rgba(255,77,79,.35); }

.ico {
  width: 28px; height: 28px;
  border-radius: 6px;
  color: var(--tx-3);
  font-size: 14px;
  background: transparent;
  border: 1px solid var(--line-2);
  cursor: pointer;
  transition: color .12s, background .12s;
}
.ico:hover:not(:disabled) { color: var(--danger); background: rgba(255,77,79,.08); }
.ico:disabled { opacity: .35; cursor: not-allowed; }

/* charts */
.charts { display: flex; flex-direction: column; gap: 18px; }
.chart-card {
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
  padding: 18px 22px;
}
.chart-head h3 { font-size: 16px; font-weight: 500; margin-bottom: 4px; }
.chart-svg { display: block; width: 100%; height: auto; max-width: 100%; margin-top: 8px; }
.chart-empty { color: var(--tx-3); padding: 40px 0; text-align: center; }
.legend {
  list-style: none; padding: 0; margin: 12px 0 0;
  display: flex; flex-wrap: wrap; gap: 12px;
  font-size: 12px; color: var(--tx-2);
}
.legend li { display: flex; align-items: center; gap: 6px; }
.legend-dot { width: 10px; height: 10px; border-radius: 999px; display: inline-block; }
.legend em { color: var(--tx-3); font-style: normal; font-size: 11px; }

/* logs table */
.logs-table {
  width: 100%; border-collapse: collapse;
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
  overflow: hidden;
  font-size: 13px;
  margin-top: 16px;
}
.logs-table th, .logs-table td {
  padding: 10px 14px;
  border-bottom: 1px solid var(--line-1);
}
.logs-table th {
  text-align: left;
  background: rgba(255,255,255,.02);
  font-size: 10.5px; letter-spacing: .14em; color: var(--tx-3);
  font-weight: 500; text-transform: uppercase;
}
.logs-table td.num { text-align: right; font-family: 'JetBrains Mono', ui-monospace, monospace; }
.muted-cell { color: var(--tx-3); font-size: 12.5px; }

/* builder modal */
.overlay {
  position: fixed; inset: 0; z-index: 60;
  display: grid; place-items: center;
  background: rgba(0,0,0,.65);
  backdrop-filter: blur(6px);
  padding: 24px;
}
.builder {
  width: 100%;
  max-width: 720px;
  max-height: 90vh;
  background: var(--bg-1);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
  display: flex; flex-direction: column;
  overflow: hidden;
}
.builder-head {
  display: flex; justify-content: space-between; align-items: center;
  padding: 16px 22px;
  border-bottom: 1px solid var(--line-2);
}
.builder-head .ico { font-size: 22px; }
.builder-body {
  padding: 20px 22px;
  display: flex; flex-direction: column; gap: 16px;
  overflow-y: auto;
}
.builder-body .field input,
.builder-body .field textarea {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid var(--line-2);
  background: var(--bg-input);
  color: var(--tx-1);
  border-radius: var(--r-sm);
  outline: none;
  font: inherit;
  font-size: 13.5px;
}
.builder-body .field input:focus { border-color: var(--tx-2); }
.builder-group {
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-sm);
  padding: 12px 14px;
  display: flex; flex-direction: column; gap: 8px;
}
.builder-ex {
  display: grid;
  grid-template-columns: 38px 2fr 60px 60px 1.5fr 30px;
  gap: 8px;
  align-items: center;
}
.builder-ex input {
  padding: 8px 10px;
  border: 1px solid var(--line-2);
  background: var(--bg-input);
  color: var(--tx-1);
  border-radius: var(--r-sm);
  outline: none;
  font-size: 12.5px;
}
.builder-ex input:focus { border-color: var(--tx-2); }
.ex-tag {
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  font-size: 11.5px; color: var(--tx-3);
  text-align: center;
}
.builder-foot {
  display: flex; justify-content: flex-end; gap: 10px;
  padding: 14px 22px;
  border-top: 1px solid var(--line-2);
  background: rgba(0,0,0,.25);
}
</style>
