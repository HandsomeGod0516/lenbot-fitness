<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, defineComponent, h } from 'vue'

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
/** 紀錄頁的群組單位：對應 menu 的 position
 *  - lanes.length === 1 → 單一動作 (依序 set 1, 2, 3...)
 *  - lanes.length >= 2 → superset (依 round 交錯：A1→B1, A2→B2)
 */
interface LogGroup {
  position: number
  lanes: LogPerExercise[]
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
const tab = ref<'menus' | 'log' | 'stats' | 'feed'>('menus')

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

// =========================================================================
// 紀錄狀態 (新版：逐 set 存檔、resume in-progress)
// =========================================================================
interface InProgressSet {
  id: number
  exerciseId: number
  exerciseName: string
  setNumber: number
  weightKg: number | null
  reps: number | null
}

const logId = ref<number | null>(null)                  // 目前 in-progress log
const logMenuLoaded = ref<MenuDetail | null>(null)      // 該 log 的菜單詳細 (有 groups)
const logSets = ref<InProgressSet[]>([])                // 該 log 已存的所有 set
const currentPosition = ref(1)                          // 使用者目前停在哪個 position
const activeLaneIdx = ref(0)                            // group 內哪一個 lane (動作) 是 active
const activeField = ref<'weight' | 'reps'>('weight')    // keypad 寫到哪個 tile
const inputW = ref('')                                  // weight 輸入緩衝
const inputR = ref('')                                  // reps 輸入緩衝
// 下一次按數字時要先清空當前欄位
// (點 tile / 切欄位 / 剛 confirm 完，準備重新輸入而不是接著敲)
const replaceOnNextDigit = ref(false)
const logNotes = ref('')                                // 整次訓練備註

const currentGroup = computed(() => {
  return logMenuLoaded.value?.groups.find(g => g.position === currentPosition.value) || null
})
const currentLane = computed(() => {
  return currentGroup.value?.exercises[activeLaneIdx.value] || null
})
const positionsCount = computed(() => logMenuLoaded.value?.groups.length || 0)
const positions = computed(() => logMenuLoaded.value?.groups.map(g => g.position).sort((a, b) => a - b) || [])
const currentPositionIdx = computed(() => positions.value.indexOf(currentPosition.value))
const hasNextPosition = computed(() => currentPositionIdx.value >= 0 && currentPositionIdx.value < positions.value.length - 1)
const hasPrevPosition = computed(() => currentPositionIdx.value > 0)

const setsForCurrentLane = computed(() => {
  if (!currentLane.value) return []
  return logSets.value
    .filter(s => s.exerciseId === currentLane.value!.id)
    .sort((a, b) => a.setNumber - b.setNumber)
})
const totalVolumeForLane = computed(() => {
  return setsForCurrentLane.value.reduce((sum, s) =>
    sum + (Number(s.weightKg) || 0) * (s.reps || 0), 0,
  )
})

// 統計
const stats = ref<StatRow[]>([])
const myLogs = ref<MyLogRow[]>([])

// feed (訓練留言板)
interface FeedRow {
  id: number
  user_id: number
  username: string
  display_name: string | null
  menu_id: number | null
  menu_name: string | null
  notes: string
  performed_at: string
  completed_at: string
  set_count: number
  total_volume: string | null
  exercise_count: number
}
const feed = ref<FeedRow[]>([])

async function loadFeed() {
  loading.value = true
  errorMsg.value = ''
  try {
    const r = await fetch('/api/fitness/feed', { credentials: 'include' })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    const d = await r.json()
    feed.value = d.feed
  } catch (e: any) { errorMsg.value = e?.message || '載入失敗' }
  finally { loading.value = false }
}

function relTime(iso: string): string {
  const sec = (Date.now() - new Date(iso).getTime()) / 1000
  if (sec < 60) return '剛剛'
  if (sec < 3600) return `${Math.floor(sec / 60)} 分鐘前`
  if (sec < 86400) return `${Math.floor(sec / 3600)} 小時前`
  if (sec < 86400 * 7) return `${Math.floor(sec / 86400)} 天前`
  const d = new Date(iso)
  return `${d.getFullYear()}/${d.getMonth() + 1}/${d.getDate()}`
}

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

async function deleteMenu(id: number, name: string) {
  const typed = prompt(`刪除菜單「${name}」？歷史紀錄會保留（menu_id 清空），但菜單本身會消失。\n\n請輸入菜單名稱以確認刪除：`)
  if (typed === null) return
  if (typed.trim() !== name) { errorMsg.value = '名稱不符，已取消刪除'; return }
  try {
    const r = await fetch(`/api/admin/fitness/menus/${id}`, { method: 'DELETE', credentials: 'include' })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    menus.value = menus.value.filter(m => m.id !== id)
    if (menuDetail.value?.menu.id === id) menuDetail.value = null
    okMsg.value = `✓ 已刪除「${name}」，歷史紀錄保留`
  } catch (e: any) { errorMsg.value = e?.message || '刪除失敗' }
}

// ---- builder ----
function addSlot(position: number) {
  const lastSlot = Math.max(...draft.value.exercises.filter(e => e.position === position).map(e => e.slot), 0)
  draft.value.exercises.push({
    name: '', position, slot: lastSlot + 1, target_sets: '', target_reps: '', notes: '',
  })
}

// 把 draft.exercises 依 position 分群，保留每筆原本的 index 給 v-model 用
const groupedDraft = computed(() => {
  const positions = [...new Set(draft.value.exercises.map(e => e.position))].sort((a, b) => a - b)
  return positions.map(pos => ({
    position: pos,
    items: draft.value.exercises
      .map((ex, idx) => ({ idx, ex }))
      .filter(item => item.ex.position === pos),
  }))
})
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

// ---- in-progress log lifecycle ----

/** 開始紀錄新菜單。如果已有 in-progress 直接接續編輯 */
async function startLog(menuId: number) {
  errorMsg.value = ''
  okMsg.value = ''
  try {
    const r = await fetch('/api/fitness/logs', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({ menuId }),
    })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    const d = await r.json()
    if (d.resumed) {
      okMsg.value = '已接續未完成的訓練 (請先完成或新開另一筆)'
    }
    await resumeInProgress()
    tab.value = 'log'
  } catch (e: any) { errorMsg.value = e?.message || '開始失敗' }
}

/** 從 server 取目前 in-progress log，更新本地狀態。回傳是否有 in-progress */
async function resumeInProgress(): Promise<boolean> {
  try {
    const r = await fetch('/api/fitness/in-progress', { credentials: 'include' })
    if (!r.ok) return false
    const d = await r.json()
    if (!d.log) {
      logId.value = null
      logMenuLoaded.value = null
      logSets.value = []
      return false
    }
    logId.value = d.log.id
    currentPosition.value = d.log.current_position || 1
    logNotes.value = d.log.notes || ''
    if (d.log.menu_id) {
      const mr = await fetch(`/api/fitness/menus/${d.log.menu_id}`, { credentials: 'include' })
      if (mr.ok) logMenuLoaded.value = await mr.json()
    }
    logSets.value = (d.sets || []).map((s: any) => ({
      id: s.id,
      exerciseId: s.exercise_id,
      exerciseName: s.exercise_name,
      setNumber: s.set_number,
      weightKg: s.weight_kg !== null ? Number(s.weight_kg) : null,
      reps: s.reps,
    }))
    activeLaneIdx.value = 0
    activeField.value = 'weight'
    inputW.value = ''
    inputR.value = ''
    return true
  } catch { return false }
}

// ---- keypad ----
function setActiveField(f: 'weight' | 'reps') {
  activeField.value = f
  replaceOnNextDigit.value = true       // 點 tile = 準備重新輸入
}
function selectLane(idx: number) {
  activeLaneIdx.value = idx
  activeField.value = 'weight'
  inputW.value = ''
  inputR.value = ''
  replaceOnNextDigit.value = false
}

function pressDigit(d: string) {
  if (replaceOnNextDigit.value) {
    if (activeField.value === 'weight') inputW.value = ''
    else inputR.value = ''
    replaceOnNextDigit.value = false
  }
  if (activeField.value === 'weight') {
    if (inputW.value === '0') inputW.value = d
    else if (inputW.value.length < 6) inputW.value += d
  } else {
    if (inputR.value === '0') inputR.value = d
    else if (inputR.value.length < 4) inputR.value += d
  }
}
function pressDot() {
  if (activeField.value !== 'weight') return
  if (replaceOnNextDigit.value) {
    inputW.value = ''
    replaceOnNextDigit.value = false
  }
  if (!inputW.value.includes('.')) inputW.value = (inputW.value || '0') + '.'
}
function pressBackspace() {
  replaceOnNextDigit.value = false      // 開始修改 = 不再 replace
  if (activeField.value === 'weight') inputW.value = inputW.value.slice(0, -1)
  else inputR.value = inputR.value.slice(0, -1)
}
function pressClear() {
  if (activeField.value === 'weight') inputW.value = ''
  else inputR.value = ''
  replaceOnNextDigit.value = false
}

async function pressConfirm() {
  if (!logId.value || !currentLane.value) return
  const w = inputW.value ? parseFloat(inputW.value) : null
  const r = inputR.value ? parseInt(inputR.value) : null
  if (w === null && r === null) {
    errorMsg.value = '請至少輸入重量或次數'
    return
  }
  errorMsg.value = ''
  try {
    const res = await fetch(`/api/fitness/logs/${logId.value}/sets`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({
        exerciseId: currentLane.value.id,
        exerciseName: currentLane.value.name,
        weightKg: w,
        reps: r,
      }),
    })
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    const d = await res.json()
    logSets.value.push({
      id: d.id,
      exerciseId: d.exerciseId,
      exerciseName: d.exerciseName,
      setNumber: d.setNumber,
      weightKg: d.weightKg,
      reps: d.reps,
    })
    // 下一組常用同樣重量 → 重量留著、reps 清空並 focus 過去；
    // 但若使用者直接想換重量，再按數字會把留著的重量也清掉。
    inputR.value = ''
    activeField.value = 'reps'
    replaceOnNextDigit.value = true
  } catch (e: any) { errorMsg.value = e?.message || '存檔失敗' }
}

// ---- per-set delete ----
async function deleteRecordedSet(setId: number) {
  if (!logId.value) return
  if (!confirm('刪除這組？')) return
  try {
    await fetch(`/api/fitness/logs/${logId.value}/sets/${setId}`, {
      method: 'DELETE', credentials: 'include',
    })
    logSets.value = logSets.value.filter(s => s.id !== setId)
  } catch (e: any) { errorMsg.value = e?.message || '刪除失敗' }
}

// ---- position navigation ----
async function patchLog(payload: any) {
  if (!logId.value) return
  await fetch(`/api/fitness/logs/${logId.value}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',
    body: JSON.stringify(payload),
  })
}

async function saveNotes() {
  if (!logId.value) return
  await patchLog({ notes: logNotes.value })
}
async function nextPosition() {
  if (!hasNextPosition.value) return
  const newPos = positions.value[currentPositionIdx.value + 1]
  await patchLog({ currentPosition: newPos })
  currentPosition.value = newPos
  selectLane(0)
}
async function prevPosition() {
  if (!hasPrevPosition.value) return
  const newPos = positions.value[currentPositionIdx.value - 1]
  await patchLog({ currentPosition: newPos })
  currentPosition.value = newPos
  selectLane(0)
}

async function finishLog() {
  if (!logId.value) return
  if (!confirm('完成這次訓練？完成後不能再加組。')) return
  try {
    await fetch(`/api/fitness/logs/${logId.value}/finish`, {
      method: 'POST', credentials: 'include',
    })
    okMsg.value = '✓ 訓練已完成'
    logId.value = null
    logMenuLoaded.value = null
    logSets.value = []
    await loadStats()
    tab.value = 'stats'
  } catch (e: any) { errorMsg.value = e?.message || '完成失敗' }
}

async function cancelLog() {
  if (!logId.value) return
  if (!confirm('放棄這次訓練？整筆 log 跟已記錄的組都會被刪掉、無法復原。')) return
  try {
    await fetch(`/api/fitness/logs/${logId.value}`, {
      method: 'DELETE', credentials: 'include',
    })
    logId.value = null
    logMenuLoaded.value = null
    logSets.value = []
    okMsg.value = '✓ 已放棄並刪除這次訓練'
    tab.value = 'menus'
  } catch (e: any) { errorMsg.value = e?.message || '刪除失敗' }
}

// ---- stats ----
// 下拉選單：動作 + 使用者，'__ALL__' 都是顯示全部 (預設都顯示)
const selectedExercise = ref<string>('__ALL__')
const selectedUser = ref<string>('__ALL__')

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

/** stats 裡所有 user 的清單 (給下拉用) */
const allUsers = computed(() => {
  const map = new Map<number, { id: number; name: string }>()
  for (const r of stats.value) {
    if (!map.has(r.user_id)) {
      map.set(r.user_id, { id: r.user_id, name: r.display_name || r.username })
    }
  }
  return Array.from(map.values()).sort((a, b) => a.name.localeCompare(b.name))
})

const visibleCharts = computed(() => {
  let list = chartsByExercise.value
  if (selectedExercise.value !== '__ALL__') {
    list = list.filter(c => c.exerciseName === selectedExercise.value)
  }
  if (selectedUser.value !== '__ALL__') {
    const uid = Number(selectedUser.value)
    list = list
      .map(c => ({ exerciseName: c.exerciseName, series: c.series.filter(s => s.userId === uid) }))
      .filter(c => c.series.length > 0)
  }
  return list
})

// 點圖表的某個點 → 顯示該日該動作所有 user 的詳細 sets
interface DaySetRow {
  log_id: number
  user_id: number
  username: string
  display_name: string | null
  set_number: number
  weight_kg: string | null
  reps: number | null
  exercise_name: string
  performed_at: string
}
interface DayDetailGroup {
  username: string
  display_name: string | null
  sets: DaySetRow[]
}
const dayDetailFor = ref<{ day: string; exerciseName: string; userId: number | null; userName: string | null } | null>(null)
const dayDetailSets = ref<DaySetRow[]>([])
const dayDetailLoading = ref(false)

async function openDayDetail(exerciseName: string, day: string, userId?: number, userName?: string) {
  // 防呆：day 可能來自 chart 點點，slice 取前 10 字確保 YYYY-MM-DD
  day = String(day).slice(0, 10)
  dayDetailFor.value = { day, exerciseName, userId: userId ?? null, userName: userName ?? null }
  dayDetailSets.value = []
  dayDetailLoading.value = true
  try {
    let url = `/api/fitness/sets?day=${encodeURIComponent(day)}&exercise=${encodeURIComponent(exerciseName)}`
    if (userId) url += `&user=${userId}`
    const r = await fetch(url, { credentials: 'include' })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    const d = await r.json()
    dayDetailSets.value = d.sets || []
  } catch (e: any) { errorMsg.value = e?.message || '載入失敗' }
  finally { dayDetailLoading.value = false }
}

function closeDayDetail() {
  dayDetailFor.value = null
  dayDetailSets.value = []
}

const dayDetailGroups = computed<DayDetailGroup[]>(() => {
  const map = new Map<string, DayDetailGroup>()
  for (const s of dayDetailSets.value) {
    const key = String(s.user_id)
    let g = map.get(key)
    if (!g) {
      g = { username: s.username, display_name: s.display_name, sets: [] }
      map.set(key, g)
    }
    g.sets.push(s)
  }
  return Array.from(map.values())
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
  props: {
    data: { type: Object as any, required: true },
    onPointClick: { type: Function as any, default: null },
  },
  setup(props: any) {
    // RWD：用 ResizeObserver 追容器寬，viewBox 跟著實際寬同步，文字字體就維持實際 px 大小
    const wrapper = ref<HTMLElement | null>(null)
    const containerW = ref(720)
    let ro: ResizeObserver | null = null

    onMounted(() => {
      if (!wrapper.value) return
      ro = new ResizeObserver(entries => {
        for (const e of entries) {
          if (e.contentRect.width > 0) {
            containerW.value = Math.max(280, Math.floor(e.contentRect.width))
          }
        }
      })
      ro.observe(wrapper.value)
    })
    onUnmounted(() => { ro?.disconnect() })

    const dims = computed(() => {
      const W = containerW.value
      const isMobile = W < 480
      const H = Math.min(Math.max(W * (isMobile ? 0.7 : 0.4), 200), 280)
      const pad = isMobile
        ? { l: 38, r: 14, t: 22, b: 30 }
        : { l: 48, r: 24, t: 22, b: 36 }
      return { W, H, pad, isMobile }
    })

    const plot = computed(() => {
      const { W, H, pad } = dims.value
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
      const { W, H, pad, isMobile } = dims.value
      const fs = isMobile ? 12 : 11        // 文字大小 (viewBox 同 1:1 px)
      const fsLabel = isMobile ? 13 : 11
      const ptR = isMobile ? 5 : 4         // 資料點半徑 (手機放大方便點)
      const p = plot.value

      // wrapper div 用來給 ResizeObserver 量寬度
      if (!p) {
        return h('div', {
          ref: wrapper, class: 'chart-wrapper',
        }, [h('div', { class: 'chart-empty' }, '無資料')])
      }

      // 軸線 (y left, x bottom)
      const axes: any[] = [
        h('line', {
          x1: pad.l, y1: pad.t, x2: pad.l, y2: H - pad.b,
          stroke: 'rgba(255,255,255,.22)', 'stroke-width': 1,
        }),
        h('line', {
          x1: pad.l, y1: H - pad.b, x2: W - pad.r, y2: H - pad.b,
          stroke: 'rgba(255,255,255,.22)', 'stroke-width': 1,
        }),
        h('text', {
          x: pad.l - 4, y: pad.t - 8, 'text-anchor': 'end',
          fill: 'rgba(255,255,255,.7)', 'font-size': fsLabel, 'font-weight': 600,
        }, 'kg'),
        h('text', {
          x: W - pad.r, y: H - 6, 'text-anchor': 'end',
          fill: 'rgba(255,255,255,.7)', 'font-size': fsLabel, 'font-weight': 600,
        }, '日期'),
      ]

      // y 刻度
      const yTicks: any[] = []
      const ySteps = isMobile ? 3 : 4
      for (let i = 0; i <= ySteps; i++) {
        const v = p.yMin + (p.yMax - p.yMin) * (i / ySteps)
        const y = p.yOf(v)
        yTicks.push(h('g', { key: 'yt' + i }, [
          h('line', {
            x1: pad.l, x2: W - pad.r, y1: y, y2: y,
            stroke: 'rgba(255,255,255,.05)', 'stroke-dasharray': '2 4',
          }),
          h('line', {
            x1: pad.l - 4, x2: pad.l, y1: y, y2: y,
            stroke: 'rgba(255,255,255,.45)', 'stroke-width': 1,
          }),
          h('text', {
            x: pad.l - 6, y: y + 4, 'text-anchor': 'end',
            fill: 'rgba(255,255,255,.6)', 'font-size': fs,
          }, v.toFixed(0)),
        ]))
      }

      // x 刻度
      const xTicks: any[] = []
      const maxXTicks = isMobile ? 4 : 6
      const showEvery = Math.max(1, Math.ceil(p.days.length / maxXTicks))
      p.days.forEach((day: string, i: number) => {
        if (i % showEvery !== 0 && i !== p.days.length - 1) return
        const x = p.xOf(day)
        xTicks.push(h('g', { key: 'xt' + i }, [
          h('line', {
            x1: x, x2: x, y1: H - pad.b, y2: H - pad.b + 4,
            stroke: 'rgba(255,255,255,.45)', 'stroke-width': 1,
          }),
          h('text', {
            x, y: H - pad.b + 16, 'text-anchor': 'middle',
            fill: 'rgba(255,255,255,.6)', 'font-size': fs,
          }, day.slice(5)),
        ]))
      })

      // 折線與資料點
      const lines: any[] = []
      const points: any[] = []
      for (const s of props.data.series) {
        const sorted = [...s.points].sort((a: any, b: any) => a.day.localeCompare(b.day))
        const pathD = sorted.map((pt: any, i: number) =>
          `${i === 0 ? 'M' : 'L'} ${p.xOf(pt.day).toFixed(1)} ${p.yOf(pt.value).toFixed(1)}`,
        ).join(' ')
        const c = lineColor(s.username)
        lines.push(h('path', {
          d: pathD, fill: 'none', stroke: c,
          'stroke-width': 2, 'stroke-linecap': 'round',
        }))
        for (const pt of sorted) {
          points.push(h('circle', {
            cx: p.xOf(pt.day), cy: p.yOf(pt.value), r: ptR,
            fill: c, stroke: '#0d0d0d', 'stroke-width': 1.5,
            style: 'cursor: pointer',
            onClick: () => props.onPointClick?.(pt.day, s.userId, s.username),
          }, [h('title', null, `${s.username} · ${pt.day} · ${pt.value} kg (點看 ${s.username} 當天詳細)`)]))
        }
      }

      return h('div', {
        ref: wrapper, class: 'chart-wrapper',
      }, [
        h('svg', {
          width: '100%', height: H, class: 'chart-svg',
          viewBox: `0 0 ${W} ${H}`,
          preserveAspectRatio: 'xMidYMid meet',
        }, [...yTicks, ...xTicks, ...axes, ...lines, ...points]),
      ])
    }
  },
})

// =========================================================================
// 鍵盤操作 — 在 log tab + 正在記錄時生效
//   0-9       → pressDigit
//   .         → pressDot (只對 weight)
//   Backspace → pressBackspace
//   Enter     → pressConfirm (送出)
//   Esc / c   → pressClear
//   w / r     → 切換 weight / reps
// =========================================================================
function onGlobalKeydown(e: KeyboardEvent) {
  if (tab.value !== 'log' || !logId.value || !currentLane.value) return
  // 焦點在 input / textarea (例如備註) 時讓原生輸入生效
  const t = e.target as HTMLElement | null
  if (t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA' || t.isContentEditable)) return
  // 帶任何修飾鍵 (Cmd/Ctrl/Alt/Meta) 一概放行，不要攔截快捷鍵
  if (e.metaKey || e.ctrlKey || e.altKey) return

  const k = e.key
  if (/^[0-9]$/.test(k)) { e.preventDefault(); pressDigit(k); return }
  if (k === '.')         { e.preventDefault(); pressDot();    return }
  if (k === 'Backspace') { e.preventDefault(); pressBackspace(); return }
  if (k === 'Enter')     { e.preventDefault(); pressConfirm(); return }
  if (k === 'Escape' || k.toLowerCase() === 'c') {
    e.preventDefault(); pressClear(); return
  }
  if (k.toLowerCase() === 'w') { e.preventDefault(); setActiveField('weight'); return }
  if (k.toLowerCase() === 'r') { e.preventDefault(); setActiveField('reps');   return }
}

// =========================================================================
// boot
// =========================================================================
onMounted(async () => {
  window.addEventListener('keydown', onGlobalKeydown)
  await fetchMe()
  await loadMenus()
  // 進來自動接上未完成的訓練
  const resumed = await resumeInProgress()
  if (resumed) tab.value = 'log'
})
onUnmounted(() => {
  window.removeEventListener('keydown', onGlobalKeydown)
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
      <button :class="['tab', { active: tab === 'feed' }]" @click="() => { loadFeed(); tab = 'feed' }">
        <span class="tab-glyph">💬</span> 留言板
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
              <button v-if="isAdmin" class="btn-ghost danger small" @click="deleteMenu(m.id, m.name)">刪除</button>
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
    <section v-if="tab === 'log'" class="page log-page">
      <div v-if="!logId" class="empty">
        請先到「菜單」tab 選一份菜單，按「▶ 開始紀錄」進來。
      </div>

      <template v-else-if="!currentGroup">
        <div class="empty">菜單沒有 POSITION 資料。</div>
      </template>

      <template v-else>
        <!-- POSITION navigation -->
        <header class="pos-nav">
          <button class="ico-btn" :disabled="!hasPrevPosition" @click="prevPosition">‹</button>
          <div class="pos-label">
            <span class="kicker">POSITION</span>
            <span class="pos-num">{{ currentPositionIdx + 1 }} / {{ positionsCount }}</span>
          </div>
          <button class="ico-btn" :disabled="!hasNextPosition" @click="nextPosition">›</button>
        </header>

        <!-- Lane tabs (superset 才會有 >1 個) -->
        <div v-if="currentGroup.exercises.length > 1" class="lane-tabs">
          <button
            v-for="(ex, i) in currentGroup.exercises"
            :key="ex.id"
            :class="['lane-tab', { active: activeLaneIdx === i }]"
            @click="selectLane(i)"
          >
            <span class="tag">{{ currentGroup.position }}.{{ ex.slot }}</span>
            {{ ex.name }}
          </button>
        </div>

        <div class="log-body">
        <div class="log-left">
        <!-- 目前動作 header -->
        <div class="ex-head">
          <div>
            <p class="ex-title">{{ currentLane?.name }}</p>
            <p class="ex-sub">
              <span v-if="setsForCurrentLane.length">{{ setsForCurrentLane.length }} 組</span>
              <span v-else>尚未紀錄</span>
              <span v-if="currentLane?.target_sets || currentLane?.target_reps" class="ex-target">
                · 建議 {{ currentLane.target_sets || '?' }} × {{ currentLane.target_reps || '?' }}
              </span>
            </p>
          </div>
          <div class="ex-vol">
            <p class="vol-num">{{ totalVolumeForLane.toFixed(0) }}</p>
            <p class="vol-unit">kg total</p>
          </div>
        </div>

        <!-- 已紀錄的 sets -->
        <ul class="set-list">
          <li v-for="s in setsForCurrentLane" :key="s.id" class="set-li">
            <span class="set-li-num">{{ s.setNumber }}</span>
            <span class="set-li-body">
              <span class="num">{{ s.weightKg ?? '-' }}</span>
              <span class="lbl">kg</span>
              <span class="x">×</span>
              <span class="num">{{ s.reps ?? '-' }}</span>
              <span class="lbl">reps</span>
            </span>
            <button class="trash" @click="deleteRecordedSet(s.id)" title="刪除">🗑</button>
          </li>
          <li v-if="!setsForCurrentLane.length" class="set-li empty-li">
            這個動作還沒紀錄任何組
          </li>
        </ul>

        <!-- 動作完成 / 訓練完成 按鈕 -->
        <div class="log-actions">
          <button v-if="hasNextPosition" class="btn-primary" @click="nextPosition">
            下一個 POSITION →
          </button>
          <button v-else class="btn-primary" @click="finishLog">
            ✓ 完成訓練
          </button>
          <button class="btn-ghost danger small" @click="cancelLog">放棄</button>
        </div>
        </div> <!-- /log-left -->

        <!-- 備註 — 自動存 (失焦時 PATCH)，「完成訓練」後會出現在留言板 -->
        <label v-if="!hasNextPosition" class="notes-field">
          <span class="notes-label">📝 備註 (整次訓練)</span>
          <textarea
            v-model="logNotes"
            rows="2"
            placeholder="今天感受、配重、注意事項…完成後會出現在「留言板」"
            @blur="saveNotes"
          />
        </label>

        <!-- 數字鍵盤 -->
        <div class="log-right">
        <div class="keypad-section">
          <div class="tiles">
            <button
              :class="['tile', { active: activeField === 'weight' }]"
              @click="setActiveField('weight')"
            >
              <span class="tile-val">{{ inputW || '0' }}</span>
              <span class="tile-unit">kg</span>
            </button>
            <button
              :class="['tile', { active: activeField === 'reps' }]"
              @click="setActiveField('reps')"
            >
              <span class="tile-val">{{ inputR || '0' }}</span>
              <span class="tile-unit">reps</span>
            </button>
          </div>

          <div class="keypad">
            <button class="key" @click="pressDigit('7')">7</button>
            <button class="key" @click="pressDigit('8')">8</button>
            <button class="key" @click="pressDigit('9')">9</button>
            <button class="key fn" @click="pressBackspace">⌫</button>

            <button class="key" @click="pressDigit('4')">4</button>
            <button class="key" @click="pressDigit('5')">5</button>
            <button class="key" @click="pressDigit('6')">6</button>
            <button class="key fn" @click="pressClear">C</button>

            <button class="key" @click="pressDigit('1')">1</button>
            <button class="key" @click="pressDigit('2')">2</button>
            <button class="key" @click="pressDigit('3')">3</button>
            <button class="key confirm" @click="pressConfirm">✓</button>

            <button class="key dot" @click="pressDot">.</button>
            <button class="key zero" @click="pressDigit('0')">0</button>
          </div>
        </div>
        </div> <!-- /log-right -->
        </div> <!-- /log-body -->
      </template>
    </section>

    <!-- ==================== 大家的成長 tab ==================== -->
    <section v-if="tab === 'stats'" class="page">
      <header class="head">
        <div>
          <p class="kicker">DASHBOARD</p>
          <h2>大家的成長曲線</h2>
        </div>
        <div class="head-actions">
          <select v-model="selectedExercise" class="ex-select" :disabled="!chartsByExercise.length">
            <option value="__ALL__">所有動作 ({{ chartsByExercise.length }})</option>
            <option
              v-for="c in chartsByExercise"
              :key="c.exerciseName"
              :value="c.exerciseName"
            >{{ c.exerciseName }}</option>
          </select>
          <select v-model="selectedUser" class="ex-select" :disabled="!allUsers.length">
            <option value="__ALL__">所有使用者 ({{ allUsers.length }})</option>
            <option
              v-for="u in allUsers"
              :key="u.id"
              :value="String(u.id)"
            >{{ u.name }}</option>
          </select>
          <button class="btn-ghost" @click="loadStats">↻ 重新載入</button>
        </div>
      </header>

      <div v-if="loading && !chartsByExercise.length" class="muted">LOADING…</div>
      <div v-else-if="!chartsByExercise.length" class="empty">還沒有任何紀錄。先去記一次！</div>

      <div v-else class="charts">
        <article v-for="c in visibleCharts" :key="c.exerciseName" class="chart-card">
          <header class="chart-head">
            <h3>{{ c.exerciseName }}</h3>
            <p class="muted">點任一資料點看當天所有人的詳細 sets</p>
          </header>
          <Chart
            :data="c"
            :on-point-click="(day: string, uid: number, uname: string) => openDayDetail(c.exerciseName, day, uid, uname)"
          />
          <ul class="legend">
            <li v-for="s in c.series" :key="s.userId">
              <span class="legend-dot" :style="{ background: colorFor(s.username) }" />
              {{ s.username }} <em>{{ s.points.length }} 天</em>
            </li>
          </ul>

          <!-- 當日詳細 sets 面板 -->
          <div
            v-if="dayDetailFor && dayDetailFor.exerciseName === c.exerciseName"
            class="day-detail"
          >
            <div class="day-detail-head">
              <p class="kicker">
                {{ dayDetailFor.day }} · {{ dayDetailFor.exerciseName }}<span
                  v-if="dayDetailFor.userName"
                > · {{ dayDetailFor.userName }}</span>
              </p>
              <button class="ico-x" @click="closeDayDetail">×</button>
            </div>
            <div v-if="dayDetailLoading" class="muted">LOADING…</div>
            <div v-else-if="!dayDetailGroups.length" class="muted">當天沒有紀錄</div>
            <div v-else class="day-detail-body">
              <div v-for="g in dayDetailGroups" :key="g.username" class="day-user">
                <div class="day-user-head">
                  <span class="user-dot" :style="{ background: colorFor(g.display_name || g.username) }" />
                  <span class="user-name">{{ g.display_name || g.username }}</span>
                  <span class="user-count">{{ g.sets.length }} 組</span>
                </div>
                <ul class="day-set-list">
                  <li v-for="s in g.sets" :key="s.log_id + '-' + s.set_number">
                    <span class="dset-num">#{{ s.set_number }}</span>
                    <span class="dset-body">
                      <span class="num">{{ s.weight_kg ?? '-' }}</span>
                      <span class="lbl">kg</span>
                      <span class="x">×</span>
                      <span class="num">{{ s.reps ?? '-' }}</span>
                      <span class="lbl">reps</span>
                    </span>
                  </li>
                </ul>
              </div>
            </div>
          </div>
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

    <!-- ==================== 留言板 tab ==================== -->
    <section v-if="tab === 'feed'" class="page">
      <header class="head">
        <div>
          <p class="kicker">FEED</p>
          <h2>大家的訓練留言</h2>
        </div>
        <button class="btn-ghost" @click="loadFeed">↻ 重新載入</button>
      </header>

      <div v-if="loading && !feed.length" class="muted">LOADING…</div>
      <div v-else-if="!feed.length" class="empty">
        還沒有任何完成的訓練留言。<br>
        在「紀錄」頁完成訓練時填備註就會出現在這裡。
      </div>

      <ul v-else class="feed-list">
        <li v-for="f in feed" :key="f.id" class="feed-card">
          <div class="feed-head">
            <div class="feed-avatar">
              {{ (f.display_name || f.username).slice(0, 1).toUpperCase() }}
            </div>
            <div class="feed-meta">
              <p class="feed-user">{{ f.display_name || f.username }}</p>
              <p class="feed-time">{{ relTime(f.completed_at) }} · {{ fmtTime(f.completed_at) }}</p>
            </div>
          </div>

          <p v-if="f.menu_name" class="feed-menu">
            <span class="kicker">MENU</span> {{ f.menu_name }}
          </p>

          <p class="feed-notes">{{ f.notes }}</p>

          <ul class="feed-stats">
            <li><span class="n">{{ f.set_count }}</span><span class="lbl">組</span></li>
            <li><span class="n">{{ f.exercise_count }}</span><span class="lbl">動作</span></li>
            <li><span class="n">{{ Number(f.total_volume || 0).toFixed(0) }}</span><span class="lbl">kg total</span></li>
          </ul>
        </li>
      </ul>
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

          <template v-for="g in groupedDraft" :key="g.position">
            <div class="builder-group">
              <p class="group-label">
                <span class="kicker">POSITION {{ g.position }}</span>
                <span v-if="g.items.length > 1" class="superset-badge">
                  SUPERSET ×{{ g.items.length }}
                </span>
              </p>
              <div v-for="item in g.items" :key="item.idx" class="builder-ex">
                <span class="ex-tag">{{ g.position }}.{{ item.ex.slot }}</span>
                <input v-model="item.ex.name" type="text" placeholder="動作名稱 *" class="grow" />
                <input v-model="item.ex.target_sets" type="number" placeholder="組" class="num-input" />
                <input v-model="item.ex.target_reps" type="number" placeholder="次" class="num-input" />
                <input v-model="item.ex.notes" type="text" placeholder="備註" class="grow" />
                <button class="ico" @click="removeExercise(item.idx)" :disabled="draft.exercises.length <= 1">×</button>
              </div>
              <button class="btn-ghost tiny" @click="addSlot(g.position)">+ 加超組合到此 group</button>
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
.head { 
  display: flex; 
  justify-content: space-between; 
  align-items: flex-end; 
  flex-wrap: wrap; /* 新增：允許內容換行 */
  gap: 16px;       /* 新增：換行時的垂直間距 */
}
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

/* ==================== 紀錄頁 (RWD: 手機單欄 / 桌機左右兩欄) ==================== */
.log-page {
  display: flex;
  flex-direction: column;
  gap: 14px;
  max-width: 1100px;
  margin: 0 auto;
}

/* 主體 = ex-head + set-list + log-actions (左) + keypad (右) */
.log-body {
  display: flex;
  flex-direction: column;
  gap: 14px;
}
.log-left {
  display: flex;
  flex-direction: column;
  gap: 14px;
  min-width: 0;
}
.log-right {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

/* 桌機：左右兩欄 (鍵盤固定 380px) */
@media (min-width: 768px) {
  .log-body {
    display: grid;
    grid-template-columns: minmax(0, 1fr) 380px;
    gap: 24px;
    align-items: start;
  }
  .log-right {
    position: sticky;
    top: 16px;
  }
  .head-actions {
    width: auto;      /* 桌機不強制佔滿 100% */
  }
  .ex-select {
    min-width: 200px; /* 桌機螢幕夠寬，恢復 200px */
  }
}

/* 手機：set list 可以縮窄 */
@media (max-width: 767px) {
  .log-page { max-width: 540px; }
  .tile { height: 64px; }
  .tile-val { font-size: 26px; }
  .key { height: 52px; font-size: 20px; }
}

.pos-nav {
  display: grid;
  grid-template-columns: 48px 1fr 48px;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
}
.ico-btn {
  width: 44px; height: 44px;
  border-radius: var(--r-sm);
  background: var(--bg-input);
  border: 1px solid var(--line-2);
  color: var(--tx-1);
  font-size: 22px;
  cursor: pointer;
  transition: background .12s, color .12s;
}
.ico-btn:hover:not(:disabled) { background: var(--hover-bg); }
.ico-btn:disabled { opacity: .25; cursor: not-allowed; }
.pos-label { text-align: center; }
.pos-label .kicker { display: block; font-size: 10px; }
.pos-num {
  font-size: 18px;
  font-weight: 600;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  margin-top: 2px;
}

/* lane tabs (superset) */
.lane-tabs {
  display: flex; gap: 6px;
  flex-wrap: wrap;
  padding: 8px;
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
}
.lane-tab {
  flex: 1; min-width: 0;
  padding: 10px 12px;
  background: transparent;
  color: var(--tx-2);
  border: 1px solid var(--line-2);
  border-radius: var(--r-sm);
  font-size: 12.5px;
  text-align: left;
  cursor: pointer;
  display: flex; flex-direction: column; gap: 2px;
  transition: background .12s, border-color .12s, color .12s;
}
.lane-tab:hover { background: rgba(255,255,255,.04); color: var(--tx-1); }
.lane-tab.active {
  background: var(--tx-1);
  color: #000;
  border-color: var(--tx-1);
}
.lane-tab .tag {
  font-size: 9.5px;
  letter-spacing: .14em;
  opacity: .6;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
}
.lane-tab.active .tag { opacity: .55; }

/* 當前動作 header */
.ex-head {
  display: flex; justify-content: space-between; align-items: flex-end;
  padding: 14px 18px;
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
}
.ex-title {
  font-size: 17px; font-weight: 500; margin: 0;
}
.ex-sub {
  font-size: 12px; color: var(--tx-3); margin-top: 4px;
}
.ex-target { margin-left: 4px; }
.ex-vol { text-align: right; }
.vol-num {
  font-size: 26px; font-weight: 600; line-height: 1;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  color: var(--tx-1); margin: 0;
}
.vol-unit { font-size: 10px; color: var(--tx-3); margin-top: 4px; letter-spacing: .14em; }

/* 已紀錄 sets list */
.set-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex; flex-direction: column;
  gap: 6px;
}
.set-li {
  display: grid;
  grid-template-columns: 44px 1fr 44px;
  gap: 10px;
  align-items: center;
  padding: 8px 12px;
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-sm);
}
.set-li-num {
  width: 32px; height: 32px;
  display: grid; place-items: center;
  background: var(--bg-input);
  border-radius: 6px;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  font-size: 13px;
  color: var(--tx-2);
}
.set-li-body {
  display: flex; align-items: baseline; gap: 6px;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
}
.set-li-body .num { font-size: 17px; color: var(--tx-1); font-weight: 500; }
.set-li-body .lbl { font-size: 10px; color: var(--tx-3); letter-spacing: .1em; }
.set-li-body .x { color: var(--tx-3); margin: 0 4px; }
.trash {
  width: 34px; height: 34px;
  border-radius: 6px;
  background: transparent;
  border: 1px solid var(--line-2);
  color: var(--tx-3);
  font-size: 14px;
  cursor: pointer;
  transition: color .12s, background .12s;
}
.trash:hover { color: var(--danger); background: rgba(255,77,79,.08); }
.empty-li {
  grid-template-columns: 1fr;
  color: var(--tx-3); font-size: 12.5px;
  text-align: center;
  padding: 14px 12px;
  border-style: dashed;
}

/* 備註欄 */
.notes-field {
  display: flex; flex-direction: column;
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
  padding: 12px 16px;
}
.notes-label {
  font-size: 11px;
  letter-spacing: .14em;
  color: var(--tx-3);
  margin-bottom: 8px;
}
.notes-field textarea {
  width: 100%;
  padding: 8px 10px;
  border: 1px solid var(--line-2);
  background: var(--bg-input);
  color: var(--tx-1);
  border-radius: var(--r-sm);
  outline: none;
  resize: vertical;
  font: inherit;
  font-size: 13px;
  min-height: 60px;
  transition: border-color .15s;
}
.notes-field textarea:focus { border-color: var(--tx-2); }

/* 完成 / 下一個 POSITION 操作 */
.log-actions {
  display: flex;
  gap: 8px;
  align-items: center;
}
.log-actions .btn-primary { flex: 1; padding: 12px; }

/* 數字鍵盤區 */
.keypad-section {
  margin-top: 6px;
  display: flex; flex-direction: column;
  gap: 8px;
}
.tiles {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}
.tile {
  position: relative;
  height: 70px;
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  background: var(--bg-tile);
  border: 2px solid var(--line-2);
  border-radius: var(--r-md);
  cursor: pointer;
  transition: border-color .12s, background .12s;
  padding: 4px 8px;
}
.tile.active {
  border-color: var(--tx-1);
  background: rgba(255,255,255,.05);
}
.tile-val {
  font-size: 30px; font-weight: 500;
  color: var(--tx-1);
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  line-height: 1;
}
.tile-unit {
  font-size: 11px; color: var(--tx-3);
  margin-top: 4px;
  letter-spacing: .14em;
}

.keypad {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 6px;
}
.key {
  height: 56px;
  background: var(--bg-tile);
  color: var(--tx-1);
  border: 1px solid var(--line-2);
  border-radius: var(--r-sm);
  font-size: 22px;
  font-weight: 500;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  cursor: pointer;
  transition: background .12s, transform .05s;
}
.key:hover { background: var(--hover-bg); }
.key:active { transform: scale(.96); }
.key.fn { font-size: 18px; color: var(--tx-2); }
.key.dot { grid-column: span 1; font-size: 26px; }
.key.zero { grid-column: span 2; font-size: 22px; }
.key.confirm {
  background: var(--tx-1);
  color: #000;
  font-size: 22px;
  font-weight: 700;
  border-color: var(--tx-1);
  grid-row: span 1;
}
.key.confirm:hover { opacity: .9; }

/* superset 交錯排列 (legacy CSS — 已不用於 log tab，但保留給將來) */
.superset-hint {
  font-size: 12px; color: var(--tx-3); margin: -4px 0 14px;
}
.round-block {
  margin-bottom: 14px;
  padding: 12px 14px;
  border: 1px solid var(--line-1);
  border-radius: var(--r-sm);
  background: rgba(255, 255, 255, .015);
}
.round-label {
  display: flex; align-items: center;
  margin-bottom: 8px;
}
.round-tag {
  font-size: 10.5px; letter-spacing: .14em;
  color: var(--tx-2);
  background: rgba(255,255,255,.04);
  border: 1px solid var(--line-2);
  padding: 2px 9px;
  border-radius: 999px;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
}
.superset-grid {
  display: flex; flex-direction: column; gap: 6px;
}
.set-grid-head.superset-head {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr;
  gap: 10px;
}
.set-row.superset-row {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr;
  gap: 10px;
}
.lane-name {
  display: flex; align-items: center; gap: 8px;
  font-size: 13px;
  color: var(--tx-1);
}
.ex-tag {
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  font-size: 10.5px;
  color: var(--tx-3);
  background: var(--bg-input);
  border: 1px solid var(--line-2);
  padding: 1px 6px;
  border-radius: 4px;
}
.round-actions { display: flex; gap: 8px; margin-top: 4px; }

/* dashboard 下拉 */
.head-actions { 
  display: flex; 
  gap: 10px; 
  align-items: center; 
  flex-wrap: wrap; /* 新增：允許按鈕與選單換行 */
  width: 100%;     /* 新增：手機版時佔滿整列 */
}
.ex-select {
  background: var(--bg-input);
  border: 1px solid var(--line-2);
  color: var(--tx-1);
  padding: 8px 12px;
  border-radius: var(--r-sm);
  font-size: 13px;
  flex: 1;           /* 新增：讓選單平均分配寬度 */
  min-width: 130px;  /* 修改：縮小最小寬度限制 */
  cursor: pointer;
  font-family: inherit;
}
.ex-select:focus { border-color: var(--tx-2); outline: none; }
.ex-select:disabled { opacity: .5; cursor: not-allowed; }

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

/* 點圖表後出現的當日詳細 panel */
.day-detail {
  margin-top: 14px;
  padding-top: 14px;
  border-top: 1px solid var(--line-2);
}
.day-detail-head {
  display: flex; justify-content: space-between; align-items: center;
  margin-bottom: 12px;
}
.day-detail-head .kicker {
  font-size: 11px; letter-spacing: .14em;
  color: var(--tx-2);
}
.ico-x {
  width: 28px; height: 28px;
  border-radius: 6px;
  background: transparent;
  color: var(--tx-3);
  border: 1px solid var(--line-2);
  font-size: 16px;
  cursor: pointer;
}
.ico-x:hover { background: rgba(255,255,255,.04); color: var(--tx-1); }

.day-detail-body { display: flex; flex-direction: column; gap: 12px; }
.day-user {
  background: var(--bg-input);
  border: 1px solid var(--line-1);
  border-radius: var(--r-sm);
  padding: 10px 12px;
}
.day-user-head {
  display: flex; align-items: center; gap: 8px;
  margin-bottom: 8px;
}
.user-dot {
  width: 10px; height: 10px; border-radius: 999px;
}
.user-name { font-size: 13px; font-weight: 500; color: var(--tx-1); flex: 1; }
.user-count {
  font-size: 11px; color: var(--tx-3);
  font-family: 'JetBrains Mono', ui-monospace, monospace;
}
.day-set-list {
  list-style: none; padding: 0; margin: 0;
  display: flex; flex-direction: column; gap: 4px;
}
.day-set-list li {
  display: grid;
  grid-template-columns: 40px 1fr;
  gap: 10px; align-items: center;
  padding: 4px 0;
}
.dset-num {
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  font-size: 11px;
  color: var(--tx-3);
}
.dset-body {
  display: flex; align-items: baseline; gap: 4px;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
}
.dset-body .num { font-size: 14px; color: var(--tx-1); font-weight: 500; }
.dset-body .lbl { font-size: 10px; color: var(--tx-3); letter-spacing: .1em; }
.dset-body .x { color: var(--tx-3); margin: 0 4px; }

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

/* ==================== 留言板 (feed) ==================== */
.feed-list {
  list-style: none; padding: 0; margin: 0;
  display: flex; flex-direction: column; gap: 12px;
}
.feed-card {
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
  padding: 16px 18px;
  display: flex; flex-direction: column; gap: 10px;
}
.feed-head {
  display: flex; align-items: center; gap: 12px;
}
.feed-avatar {
  width: 36px; height: 36px;
  border-radius: 999px;
  background: linear-gradient(135deg, hsl(220, 50%, 50%), hsl(280, 50%, 50%));
  display: grid; place-items: center;
  color: var(--tx-1);
  font-weight: 600;
  font-size: 14px;
  flex-shrink: 0;
}
.feed-meta { flex: 1; min-width: 0; }
.feed-user {
  font-size: 14px;
  font-weight: 500;
  color: var(--tx-1);
  margin: 0;
}
.feed-time {
  font-size: 11px;
  color: var(--tx-3);
  margin: 2px 0 0;
}
.feed-menu {
  font-size: 12.5px;
  color: var(--tx-2);
  margin: 0;
  padding: 6px 10px;
  background: var(--bg-input);
  border-radius: var(--r-sm);
  display: inline-flex;
  align-items: center;
  gap: 8px;
  align-self: flex-start;
}
.feed-menu .kicker {
  font-size: 10px;
}
.feed-notes {
  font-size: 14px;
  line-height: 1.7;
  color: var(--tx-1);
  white-space: pre-wrap;
  margin: 4px 0 6px;
}
.feed-stats {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex; gap: 16px;
  flex-wrap: wrap;
  padding-top: 10px;
  border-top: 1px solid var(--line-1);
}
.feed-stats li {
  display: flex; align-items: baseline; gap: 4px;
}
.feed-stats .n {
  font-size: 18px;
  font-weight: 600;
  color: var(--tx-1);
  font-family: 'JetBrains Mono', ui-monospace, monospace;
}
.feed-stats .lbl {
  font-size: 11px;
  color: var(--tx-3);
  letter-spacing: .14em;
}

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
