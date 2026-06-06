<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch, nextTick, defineComponent, h } from 'vue'

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
const editingMenuId = ref<number | null>(null)   // null = 新增；有值 = 編輯該菜單
const builderTitle = computed(() => editingMenuId.value ? '編輯菜單' : '新增菜單')
const draft = ref<Draft>({
  name: '',
  description: '',
  exercises: [{ name: '', position: 1, slot: 1, target_sets: '', target_reps: '', notes: '' }],
})

// 過往所有設定過的動作名稱 (給動作名稱欄位搜尋下拉)
const exerciseNames = ref<string[]>([])
async function loadExerciseNames() {
  try {
    const r = await fetch('/api/fitness/exercise-names', { credentials: 'include' })
    if (!r.ok) return
    const d = await r.json()
    exerciseNames.value = d.names || []
  } catch { /* ignore */ }
}

// 菜單列表的搜尋 / 篩選
const menuSearch = ref('')                 // 名稱 / 描述關鍵字
const menuCreatorFilter = ref('__ALL__')   // 建立者 id (string) 或 __ALL__
const menuCreators = computed(() => {
  const map = new Map<number, string>()
  for (const m of menus.value) map.set(m.created_by, m.created_by_username)
  return Array.from(map, ([id, username]) => ({ id, username }))
    .sort((a, b) => a.username.localeCompare(b.username))
})
const filteredMenus = computed(() => {
  const kw = menuSearch.value.trim().toLowerCase()
  const creator = menuCreatorFilter.value
  return menus.value.filter((m) => {
    if (creator !== '__ALL__' && String(m.created_by) !== creator) return false
    if (kw) {
      const hay = `${m.name} ${m.description || ''} ${m.created_by_username}`.toLowerCase()
      if (!hay.includes(kw)) return false
    }
    return true
  })
})

// 能不能編輯/刪除這份菜單：本人或 admin
function canManage(m: MenuSummary): boolean {
  return !!me.value && (me.value.id === m.created_by || me.value.role === 'admin')
}

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
const logNotes = ref('')                                // 整次訓練備註 (resume 用，DB 拉回來的暫存)

// 完成訓練 modal — 點「完成訓練」時跳出，讓使用者補備註 + 日期
const showFinishModal = ref(false)
const finishNotes = ref('')
const finishDate = ref('')   // YYYY-MM-DD

function todayYMD(): string {
  const d = new Date()
  const yyyy = d.getFullYear()
  const mm = String(d.getMonth() + 1).padStart(2, '0')
  const dd = String(d.getDate()).padStart(2, '0')
  return `${yyyy}-${mm}-${dd}`
}

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
    const r = await fetch(`/api/fitness/menus/${id}`, { method: 'DELETE', credentials: 'include' })
    if (!r.ok) {
      const err = await r.json().catch(() => ({}))
      throw new Error(err?.statusMessage || `HTTP ${r.status}`)
    }
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
  editingMenuId.value = null
  draft.value = {
    name: '', description: '',
    exercises: [{ name: '', position: 1, slot: 1, target_sets: '', target_reps: '', notes: '' }],
  }
}

/** 開新增菜單 */
function openBuilder() {
  resetDraft()
  showBuilder.value = true
}

/** 開編輯菜單：抓菜單詳細灌進 draft */
async function openEditMenu(m: MenuSummary) {
  errorMsg.value = ''
  okMsg.value = ''
  try {
    const r = await fetch(`/api/fitness/menus/${m.id}`, { credentials: 'include' })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    const detail = await r.json() as MenuDetail
    const exs: DraftExercise[] = []
    for (const g of detail.groups) {
      for (const ex of g.exercises) {
        exs.push({
          name: ex.name,
          position: g.position,
          slot: ex.slot,
          target_sets: ex.target_sets != null ? String(ex.target_sets) : '',
          target_reps: ex.target_reps != null ? String(ex.target_reps) : '',
          notes: ex.notes || '',
        })
      }
    }
    if (!exs.length) exs.push({ name: '', position: 1, slot: 1, target_sets: '', target_reps: '', notes: '' })
    draft.value = {
      name: detail.menu.name,
      description: detail.menu.description || '',
      exercises: exs,
    }
    editingMenuId.value = m.id
    showBuilder.value = true
  } catch (e: any) { errorMsg.value = e?.message || '載入失敗' }
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
  const isEdit = editingMenuId.value !== null
  try {
    const r = await fetch(
      isEdit ? `/api/fitness/menus/${editingMenuId.value}` : '/api/fitness/menus',
      {
        method: isEdit ? 'PUT' : 'POST',
        headers: { 'Content-Type': 'application/json' },
        credentials: 'include',
        body: JSON.stringify({
          name: draft.value.name.trim(),
          description: draft.value.description.trim() || null,
          exercises: exs,
        }),
      },
    )
    if (!r.ok) {
      const err = await r.json().catch(() => ({}))
      throw new Error(err?.statusMessage || `HTTP ${r.status}`)
    }
    okMsg.value = isEdit ? '✓ 菜單已更新' : '✓ 菜單已建立'
    showBuilder.value = false
    const editedId = editingMenuId.value
    resetDraft()
    await loadMenus()
    await loadExerciseNames()           // 動作清單可能有新名稱
    // 編輯中若正在看該菜單詳情，重新整理
    if (isEdit && editedId && menuDetail.value?.menu.id === editedId) {
      await openMenu(editedId)
    }
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
    // 先清掉舊菜單；菜單若已被刪除 (menu_id = NULL 或抓不到) 就維持 null，
    // 由 log tab 顯示「無法繼續」並提供放棄/完成的出口。
    logMenuLoaded.value = null
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
// 已紀錄 sets 的捲動容器 (手機版會出現滾輪) → 用來自動捲到最底
const setListRef = ref<HTMLElement | null>(null)
function scrollSetsToBottom() {
  nextTick(() => {
    const el = setListRef.value
    if (el) el.scrollTop = el.scrollHeight
  })
}

function setActiveField(f: 'weight' | 'reps') {
  activeField.value = f
  replaceOnNextDigit.value = true       // 點 tile = 準備重新輸入
  if (f === 'weight') scrollSetsToBottom()   // 開始輸入重量時，把最近的組捲到看得見
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
    // 送出後保留重量與 reps 數值，讓使用者連續記錄同樣的組數時可直接再按送出；
    // 想修改任一欄位時，按下第一個數字會自動取代舊值。
    activeField.value = 'reps'
    replaceOnNextDigit.value = true
    scrollSetsToBottom()                 // 新增一組後捲到最底，讓最新組可見
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

// 點「完成訓練」改成跳 modal：讓使用者補備註 + 補填日期 (default 今天)
function openFinishModal() {
  if (!logId.value) return
  finishNotes.value = logNotes.value || ''
  finishDate.value = todayYMD()
  showFinishModal.value = true
}

async function confirmFinish() {
  if (!logId.value) return
  errorMsg.value = ''
  try {
    const r = await fetch(`/api/fitness/logs/${logId.value}/finish`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({
        notes: finishNotes.value.trim() || null,    // 空字串視為沒填
        performedAt: finishDate.value || null,
      }),
    })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    okMsg.value = '✓ 訓練已完成'
    showFinishModal.value = false
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
// 下拉選單：菜單 + 動作 + 使用者 + 日期區間 + 指標 + 排序
// '__ALL__' 都是顯示全部 (預設都顯示)
const selectedExercise = ref<string>('__ALL__')
const selectedUser = ref<string>('__ALL__')
const selectedMenu = ref<string>('__ALL__')
type MetricType = 'max_weight' | 'total_volume'
type SortMode = 'name' | 'recent' | 'volume'
const dateFrom = ref<string>('')   // YYYY-MM-DD，空字串 = 不限起日
const dateTo = ref<string>('')     // YYYY-MM-DD，空字串 = 不限迄日
const selectedMetric = ref<MetricType>('max_weight')
const selectedSort = ref<SortMode>('name')

function clearDateRange() {
  dateFrom.value = ''
  dateTo.value = ''
}

// 菜單 → 該菜單包含的動作名稱 (用 Set，用於過濾 charts)
const menuExerciseNames = ref<Map<number, Set<string>>>(new Map())
async function ensureMenuExercises(menuId: number) {
  if (menuExerciseNames.value.has(menuId)) return
  try {
    const r = await fetch(`/api/fitness/menus/${menuId}`, { credentials: 'include' })
    if (!r.ok) return
    const d = await r.json() as MenuDetail
    const names = new Set<string>()
    for (const g of d.groups) for (const ex of g.exercises) names.add(ex.name)
    menuExerciseNames.value.set(menuId, names)
    // 觸發 reactivity (Map.set 不會自動觸發)
    menuExerciseNames.value = new Map(menuExerciseNames.value)
  } catch { /* ignore */ }
}
watch(selectedMenu, (v) => {
  if (v !== '__ALL__') ensureMenuExercises(Number(v))
})
const metricLabel = computed(() => selectedMetric.value === 'max_weight' ? 'kg' : 'vol')
const metricCaption = computed(() => selectedMetric.value === 'max_weight' ? '最大重量' : '訓練總量')

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
  const from = dateFrom.value || null
  const to = dateTo.value || null
  const byEx = new Map<string, Map<number, ChartUserSeries>>()
  for (const r of stats.value) {
    if (from && r.day < from) continue
    if (to && r.day > to) continue
    const v = selectedMetric.value === 'max_weight'
      ? (r.max_weight ? Number(r.max_weight) : null)
      : (r.total_volume ? Number(r.total_volume) : null)
    if (v === null || isNaN(v)) continue
    let users = byEx.get(r.exercise_name)
    if (!users) { users = new Map(); byEx.set(r.exercise_name, users) }
    let series = users.get(r.user_id)
    if (!series) {
      series = { userId: r.user_id, username: r.display_name || r.username, points: [] }
      users.set(r.user_id, series)
    }
    series.points.push({ day: r.day, value: v })
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

// 每張 chart 的「最新日期」與「總資料點數」(給排序 / 過濾用)
function chartLastDay(c: ChartData): string {
  let last = ''
  for (const s of c.series) {
    for (const p of s.points) if (p.day > last) last = p.day
  }
  return last
}
function chartPointCount(c: ChartData): number {
  let n = 0
  for (const s of c.series) n += s.points.length
  return n
}

const visibleCharts = computed(() => {
  let list = chartsByExercise.value
  if (selectedMenu.value !== '__ALL__') {
    const names = menuExerciseNames.value.get(Number(selectedMenu.value))
    if (names) list = list.filter(c => names.has(c.exerciseName))
    else list = []   // 菜單動作清單還沒載入完，先空白
  }
  if (selectedExercise.value !== '__ALL__') {
    list = list.filter(c => c.exerciseName === selectedExercise.value)
  }
  if (selectedUser.value !== '__ALL__') {
    const uid = Number(selectedUser.value)
    list = list
      .map(c => ({ exerciseName: c.exerciseName, series: c.series.filter(s => s.userId === uid) }))
      .filter(c => c.series.length > 0)
  }
  // 排序
  const sortMode = selectedSort.value
  list = [...list].sort((a, b) => {
    if (sortMode === 'recent') {
      return chartLastDay(b).localeCompare(chartLastDay(a))
    }
    if (sortMode === 'volume') {
      return chartPointCount(b) - chartPointCount(a)
    }
    return a.exerciseName.localeCompare(b.exerciseName)
  })
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
    yLabel: { type: String, default: 'kg' },
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
        }, props.yLabel),
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
  await loadExerciseNames()
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
        <button class="btn-primary" @click="openBuilder">+ 新增菜單</button>
      </header>

      <!-- 搜尋 / 篩選 -->
      <div v-if="menus.length" class="menu-filters">
        <input
          v-model="menuSearch"
          type="search"
          class="ex-select menu-search"
          placeholder="🔍 搜尋菜單名稱 / 描述 / 建立者"
        />
        <label class="filter-field">
          <span class="filter-label">建立者</span>
          <select v-model="menuCreatorFilter" class="ex-select">
            <option value="__ALL__">所有人 ({{ menuCreators.length }})</option>
            <option v-for="c in menuCreators" :key="c.id" :value="String(c.id)">{{ c.username }}</option>
          </select>
        </label>
        <button
          v-if="menuSearch || menuCreatorFilter !== '__ALL__'"
          class="btn-ghost small"
          @click="() => { menuSearch = ''; menuCreatorFilter = '__ALL__' }"
        >× 清除</button>
      </div>

      <div v-if="loading && !menus.length" class="muted">LOADING…</div>
      <div v-else-if="!menus.length" class="empty">尚無菜單，點上方「+ 新增菜單」開始</div>
      <div v-else-if="!filteredMenus.length" class="empty">找不到符合條件的菜單，試著放寬搜尋。</div>

      <div v-else class="menu-list">
        <article v-for="m in filteredMenus" :key="m.id" class="menu-card">
          <div class="menu-head">
            <div>
              <h3>{{ m.name }}</h3>
              <p class="muted">{{ m.description || '—' }}</p>
              <p class="meta">建立者 {{ m.created_by_username }} · {{ m.exercise_count }} 個動作 · {{ fmtTime(m.created_at) }}</p>
            </div>
            <div class="menu-actions">
              <button class="btn-ghost" @click="openMenu(m.id)">查看</button>
              <button class="btn-primary" @click="startLog(m.id)">▶ 開始紀錄</button>
              <button v-if="canManage(m)" class="btn-ghost small" @click="openEditMenu(m)">編輯</button>
              <button v-if="canManage(m)" class="btn-ghost danger small" @click="deleteMenu(m.id, m.name)">刪除</button>
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

      <template v-else-if="!logMenuLoaded || !currentGroup">
        <div class="orphan-log">
          <p class="orphan-title">⚠ 這次訓練無法繼續</p>
          <p class="muted">
            {{ !logMenuLoaded
              ? '這次訓練使用的菜單已被刪除，找不到動作內容。'
              : '這份菜單沒有任何動作資料。' }}
          </p>
          <p v-if="logSets.length" class="muted">
            已經記錄 {{ logSets.length }} 組，可以「完成並保存」留到歷史紀錄，或直接放棄。
          </p>
          <div class="orphan-actions">
            <button v-if="logSets.length" class="btn-primary" @click="openFinishModal">✓ 完成並保存</button>
            <button class="btn-ghost danger" @click="cancelLog">🗑 放棄這次訓練</button>
          </div>
        </div>
      </template>

      <template v-else>
        <!-- POSITION navigation (compact) -->
        <header class="pos-nav">
          <button class="ico-btn" :disabled="!hasPrevPosition" @click="prevPosition">‹</button>
          <div class="pos-label">
            <p class="ex-sub">
              <span v-if="currentGroup.exercises.length == 1">{{currentGroup.exercises[0].name}}</span>
              <span v-if="setsForCurrentLane.length">{{ setsForCurrentLane.length }} 組</span>
              <span v-if="currentLane?.target_sets || currentLane?.target_reps" class="ex-target">
                建議 {{ currentLane.target_sets || '?' }} × {{ currentLane.target_reps || '?' }}
              </span>
            </p>
            <span class="pos-num">{{ currentPositionIdx + 1 }} / {{ positionsCount }}</span>
            <p class="vol-num">{{ totalVolumeForLane.toFixed(0) }}</p>
            <p class="vol-unit">kg total</p>
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
        <div class="log-left" ref="setListRef">
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
        </div> <!-- /log-left -->

        <!-- 數字鍵盤 -->
        <div class="log-right">
        <!-- 訓練完成按鈕 (跳 modal 補備註 + 日期) -->
        <div class="log-actions">
          <button class="btn-primary" @click="openFinishModal">
            ✓ 完成訓練
          </button>
        </div>
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
            <button class="key cancel" @click="cancelLog" title="放棄這次訓練">放棄</button>
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
          <label class="filter-field">
            <span class="filter-label">菜單</span>
            <select v-model="selectedMenu" class="ex-select" :disabled="!menus.length">
              <option value="__ALL__">所有菜單 ({{ menus.length }})</option>
              <option
                v-for="m in menus"
                :key="m.id"
                :value="String(m.id)"
              >{{ m.name }}</option>
            </select>
          </label>
          <label class="filter-field">
            <span class="filter-label">動作</span>
            <select v-model="selectedExercise" class="ex-select" :disabled="!chartsByExercise.length">
              <option value="__ALL__">所有動作 ({{ chartsByExercise.length }})</option>
              <option
                v-for="c in chartsByExercise"
                :key="c.exerciseName"
                :value="c.exerciseName"
              >{{ c.exerciseName }}</option>
            </select>
          </label>
          <label class="filter-field">
            <span class="filter-label">使用者</span>
            <select v-model="selectedUser" class="ex-select" :disabled="!allUsers.length">
              <option value="__ALL__">所有使用者 ({{ allUsers.length }})</option>
              <option
                v-for="u in allUsers"
                :key="u.id"
                :value="String(u.id)"
              >{{ u.name }}</option>
            </select>
          </label>
          <div class="filter-field date-range-field">
            <span class="filter-label">日期區間</span>
            <div class="date-range">
              <input
                v-model="dateFrom"
                type="date"
                class="ex-select date-input"
                :max="dateTo || undefined"
                aria-label="起日"
              />
              <span class="date-sep">–</span>
              <input
                v-model="dateTo"
                type="date"
                class="ex-select date-input"
                :min="dateFrom || undefined"
                aria-label="迄日"
              />
            </div>
          </div>
          <label class="filter-field">
            <span class="filter-label">指標</span>
            <select v-model="selectedMetric" class="ex-select">
              <option value="max_weight">最大重量 (kg)</option>
              <option value="total_volume">訓練總量 (vol)</option>
            </select>
          </label>
          <label class="filter-field">
            <span class="filter-label">排序</span>
            <select v-model="selectedSort" class="ex-select">
              <option value="name">名稱</option>
              <option value="recent">最近活動</option>
              <option value="volume">最多資料點</option>
            </select>
          </label>
          <button
            v-if="dateFrom || dateTo"
            class="btn-ghost small"
            @click="clearDateRange"
            title="清除日期區間"
          >× 清除日期</button>
          <button class="btn-ghost" @click="loadStats">↻ 重新載入</button>
        </div>
      </header>

      <div v-if="loading && !chartsByExercise.length" class="muted">LOADING…</div>
      <div v-else-if="!chartsByExercise.length" class="empty">還沒有任何紀錄。先去記一次！</div>
      <div v-else-if="!visibleCharts.length" class="empty">目前篩選條件下沒有資料，試著放寬條件。</div>

      <div v-else class="charts">
        <article v-for="c in visibleCharts" :key="c.exerciseName" class="chart-card">
          <header class="chart-head">
            <h3>{{ c.exerciseName }}</h3>
            <p class="muted">{{ metricCaption }} · 點任一資料點看當天所有人的詳細 sets</p>
          </header>
          <Chart
            :data="c"
            :y-label="metricLabel"
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

    <!-- ==================== Finish modal (完成訓練 → 補備註 + 日期) ==================== -->
    <div v-if="showFinishModal" class="overlay" @click.self="showFinishModal = false">
      <div class="finish-modal">
        <header class="finish-head">
          <div>
            <p class="kicker">FINISH</p>
            <h3>完成訓練</h3>
          </div>
          <button class="ico" @click="showFinishModal = false">×</button>
        </header>
        <div class="finish-body">
          <label class="field">
            <span>訓練日期</span>
            <input v-model="finishDate" type="date" class="finish-date" />
            <small class="hint">忘記填寫的可以改成當天日期補填</small>
          </label>
          <label class="field">
            <span>備註 (選填)</span>
            <textarea
              v-model="finishNotes"
              rows="4"
              placeholder="今天感受、配重、注意事項… 留空就不會出現在留言板"
            />
          </label>
        </div>
        <footer class="finish-foot">
          <button class="btn-ghost" @click="showFinishModal = false">取消</button>
          <button class="btn-primary" @click="confirmFinish">✓ 確認完成</button>
        </footer>
      </div>
    </div>

    <!-- ==================== Builder modal ==================== -->
    <div v-if="showBuilder" class="overlay" @click.self="showBuilder = false">
      <div class="builder">
        <header class="builder-head">
          <p class="kicker">{{ editingMenuId ? 'EDIT MENU' : 'NEW MENU' }} · {{ builderTitle }}</p>
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
                <input
                  v-model="item.ex.name"
                  type="text"
                  list="exercise-name-options"
                  placeholder="動作名稱 * (可搜尋過往動作)"
                  class="grow"
                />
                <input v-model="item.ex.target_sets" type="number" placeholder="組" class="num-input" />
                <input v-model="item.ex.target_reps" type="number" placeholder="次" class="num-input" />
                <input v-model="item.ex.notes" type="text" placeholder="備註" class="grow" />
                <button class="ico" @click="removeExercise(item.idx)" :disabled="draft.exercises.length <= 1">×</button>
              </div>
              <button class="btn-ghost tiny" @click="addSlot(g.position)">+ 加超組合到此 group</button>
            </div>
          </template>

          <button class="btn-ghost" @click="addPosition">+ 加下一個 position (新動作)</button>

          <!-- 過往所有動作名稱：動作名稱欄位的搜尋下拉來源 -->
          <datalist id="exercise-name-options">
            <option v-for="n in exerciseNames" :key="n" :value="n" />
          </datalist>
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
/* touch-action: manipulation → 停掉手機「快速點兩下放大」(double-tap zoom)，
   仍保留正常捲動與雙指縮放 */
.fit { display: flex; flex-direction: column; gap: 18px; touch-action: manipulation; }

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

/* 孤兒訓練 (菜單已被刪除) 的提示 + 出口 */
.orphan-log {
  text-align: center; padding: 48px 24px;
  background: var(--bg-tile);
  border: 1px solid var(--line-3);
  border-radius: var(--r-md);
  display: flex; flex-direction: column; gap: 12px; align-items: center;
}
.orphan-title { font-size: 16px; font-weight: 600; color: var(--danger); }
.orphan-actions { display: flex; gap: 12px; flex-wrap: wrap; justify-content: center; margin-top: 8px; }

/* menu list */
.menu-filters {
  display: flex; gap: 10px; flex-wrap: wrap; align-items: flex-end;
  margin-bottom: 16px;
}
.menu-search {
  flex: 1 1 240px;
  min-width: 180px;
}
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
  width: 100%;
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

/* 手機：set list 可以縮窄、整體更緊湊 */
@media (max-width: 767px) {
  /* .log-page 鎖定為視窗高度，內部用 flex 控制 (扣掉 portal-main 手機版 topbar ~47px
     + container 上下 padding 10+16=26 + tabs+gap ~50 ≈ 130) */
  .log-page {
    max-width: 540px;
    align-self: center;
    gap: 10px;
    height: calc(100dvh - 130px);
    min-height: 0;
  }
  .log-body {
    flex: 1;
    min-height: 0;
  }
  .log-left {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
  }
  .log-right {
    flex: 0 0 auto;
  }

  /* 外層 gap 收緊 */
  .fit { gap: 10px; }

  /* tabs 變小 */
  .tab { padding: 7px 12px; font-size: 11.5px; gap: 6px; }
  .tab-glyph { font-size: 13px; }

  /* POSITION nav 收緊 */
  .pos-nav { padding: 4px 10px; gap: 8px; grid-template-columns: 30px 1fr 30px; }
  .ico-btn { width: 28px; height: 28px; font-size: 14px; }
  .pos-num { font-size: 12px; }
  .pos-label .kicker { font-size: 9px; }

  /* 動作 header 收緊 */
  .ex-head { padding: 10px 14px; }
  .ex-title { font-size: 15px; }
  .vol-num { font-size: 22px; }

  /* 紀錄頁主體間距 */
  .log-body { gap: 10px; }
  .log-left, .log-right { gap: 10px; }

  /* 動作完成按鈕 */
  .log-actions .btn-primary { padding: 10px; font-size: 12px; }

  /* set list — 手機更緊湊，一次看到更多組 */
  .set-list { gap: 2px; }
  .set-li { padding: 3px 8px; grid-template-columns: 20px 1fr 24px; gap: 7px; }
  .set-li-num { width: 20px; height: 20px; font-size: 10px; border-radius: 5px; }
  .set-li-body .num { font-size: 16px; }
  .set-li-body .lbl { font-size: 9px; }
  .set-li-body .x { font-size: 12px; margin: 0 3px; }
  .trash { width: 24px; height: 24px; font-size: 11px; }
  .empty-li { padding: 8px 12px; font-size: 11px; }

  /* tiles + keypad 收緊 */
  .keypad-section { gap: 5px; margin-top: 2px; }
  .tile { height: 40px; }
  .tile-val { font-size: 18px; }
  .tile-unit { font-size: 8px; margin-top: 1px; }
  .tiles { gap: 5px; }
  .key { height: 32px; font-size: 15px; }
  .key.cancel { font-size: 11px; }
  .keypad { gap: 4px; }
}

.pos-nav {
  display: grid;
  grid-template-columns: 34px 1fr 34px;
  align-items: center;
  gap: 10px;
  padding: 6px 12px;
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
}
.ico-btn {
  width: 30px; height: 30px;
  border-radius: var(--r-sm);
  background: var(--bg-input);
  border: 1px solid var(--line-2);
  color: var(--tx-1);
  font-size: 16px;
  cursor: pointer;
  transition: background .12s, color .12s;
  display: grid; place-items: center;
}
.ico-btn:hover:not(:disabled) { background: var(--hover-bg); }
.ico-btn:disabled { opacity: .25; cursor: not-allowed; }
.pos-label {
  display: flex; align-items: baseline; justify-content: center; gap: 8px;
}
.pos-label .kicker { font-size: 9.5px; letter-spacing: .22em; }
.pos-num {
  font-size: 13px;
  font-weight: 600;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
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

/* 已紀錄 sets list — 數字放大讓使用者一眼就看到組數 */
.set-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex; flex-direction: column;
  gap: 3px;
}
.set-li {
  display: grid;
  grid-template-columns: 24px 1fr 26px;
  gap: 8px;
  align-items: center;
  padding: 4px 10px;
  background: var(--bg-tile);
  border: 1px solid var(--line-2);
  border-radius: var(--r-sm);
}
.set-li-num {
  width: 24px; height: 24px;
  display: grid; place-items: center;
  background: var(--bg-input);
  border-radius: 6px;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  font-size: 12px;
  font-weight: 600;
  color: var(--tx-1);
}
.set-li-body {
  display: flex; align-items: baseline; gap: 6px;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
}
.set-li-body .num { font-size: 18px; color: var(--tx-1); font-weight: 600; line-height: 1; }
.set-li-body .lbl { font-size: 10px; color: var(--tx-3); letter-spacing: .1em; }
.set-li-body .x { color: var(--tx-3); margin: 0 4px; font-size: 14px; }
.trash {
  width: 26px; height: 26px;
  border-radius: 6px;
  background: transparent;
  border: 1px solid var(--line-2);
  color: var(--tx-3);
  font-size: 13px;
  cursor: pointer;
  transition: color .12s, background .12s;
}
.trash:hover { color: var(--danger); background: rgba(255,77,79,.08); }
.empty-li {
  grid-template-columns: 1fr;
  color: var(--tx-3); font-size: 12.5px;
  text-align: center;
  padding: 12px;
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
  margin-top: 4px;
  display: flex; flex-direction: column;
  gap: 6px;
}
.tiles {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px;
}
.tile {
  position: relative;
  height: 46px;
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  background: var(--bg-tile);
  border: 2px solid var(--line-2);
  border-radius: var(--r-md);
  cursor: pointer;
  touch-action: manipulation;   /* 連點不放大 */
  transition: border-color .12s, background .12s;
  padding: 4px 8px;
}
.tile.active {
  border-color: var(--tx-1);
  background: rgba(255,255,255,.05);
}
.tile-val {
  font-size: 21px; font-weight: 500;
  color: var(--tx-1);
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  line-height: 1;
}
.tile-unit {
  font-size: 9px; color: var(--tx-3);
  margin-top: 2px;
  letter-spacing: .14em;
}

.keypad {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 5px;
}
.key {
  height: 36px;
  background: var(--bg-tile);
  color: var(--tx-1);
  border: 1px solid var(--line-2);
  border-radius: var(--r-sm);
  font-size: 18px;
  font-weight: 500;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  cursor: pointer;
  touch-action: manipulation;   /* 連點不放大 */
  transition: background .12s, transform .05s;
}
.key:hover { background: var(--hover-bg); }
.key:active { transform: scale(.96); }
.key.fn { font-size: 16px; color: var(--tx-2); }
.key.dot { grid-column: span 1; font-size: 22px; }
.key.zero { grid-column: span 2; font-size: 18px; }
.key.confirm {
  background: var(--tx-1);
  color: #000;
  font-size: 18px;
  font-weight: 700;
  border-color: var(--tx-1);
  grid-row: span 1;
}
.key.confirm:hover { opacity: .9; }
.key.cancel {
  background: rgba(255,77,79,.08);
  color: var(--danger);
  border-color: rgba(255,77,79,.30);
  font-size: 13px;
  font-family: inherit;
  letter-spacing: .04em;
}
.key.cancel:hover { background: rgba(255,77,79,.16); }

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
  width: 100%;
}
.ex-select:focus { border-color: var(--tx-2); outline: none; }
.ex-select:disabled { opacity: .5; cursor: not-allowed; }
.date-input {
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  color-scheme: dark;        /* 讓原生日歷 picker 用暗色主題 */
  min-width: 0;              /* 允許在 flex 容器內縮小，避免擠到別的欄位 */
}

/* 日期區間：兩個 date input 共用一格欄位 */
.date-range-field { flex: 1 1 260px; min-width: 220px; }
.date-range {
  display: flex;
  align-items: center;
  gap: 6px;
}
.date-range .date-input {
  flex: 1 1 0;
  width: 100%;
  padding: 8px 8px;
  font-size: 12px;
}
.date-sep {
  color: var(--tx-3);
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  font-size: 14px;
  flex: 0 0 auto;
}
@media (min-width: 768px) {
  .date-range-field { flex: 0 0 auto; min-width: 260px; }
}

/* 每個下拉欄位包一層 label，讓上方標籤清楚 */
.filter-field {
  display: flex;
  flex-direction: column;
  gap: 4px;
  flex: 1 1 150px;
  min-width: 130px;
}
.filter-label {
  font-size: 10px;
  letter-spacing: .14em;
  color: var(--tx-3);
  text-transform: uppercase;
}
@media (min-width: 768px) {
  .filter-field { flex: 0 0 auto; min-width: 150px; }
}

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
  background: var(--bg-elev);
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

/* ==================== Finish modal ==================== */
.finish-modal {
  width: 100%;
  max-width: 460px;
  background: var(--bg-elev);
  border: 1px solid var(--line-2);
  border-radius: var(--r-md);
  display: flex; flex-direction: column;
  overflow: hidden;
}
.finish-head {
  display: flex; justify-content: space-between; align-items: flex-start;
  padding: 16px 22px;
  border-bottom: 1px solid var(--line-2);
}
.finish-head h3 { font-size: 18px; font-weight: 500; margin-top: 4px; }
.finish-head .ico { font-size: 22px; }
.finish-body {
  padding: 18px 22px;
  display: flex; flex-direction: column; gap: 16px;
}
.finish-body .field span {
  display: block; font-size: 10.5px; color: var(--tx-3);
  letter-spacing: .14em; margin-bottom: 6px; text-transform: uppercase;
}
.finish-body .field input,
.finish-body .field textarea {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid var(--line-2);
  background: var(--bg-input);
  color: var(--tx-1);
  border-radius: var(--r-sm);
  outline: none;
  font: inherit;
  font-size: 14px;
  resize: vertical;
  transition: border-color .15s;
}
.finish-body .field input:focus,
.finish-body .field textarea:focus { border-color: var(--tx-2); }
.finish-date { font-family: 'JetBrains Mono', ui-monospace, monospace; }
.finish-body .hint {
  display: block;
  font-size: 11px;
  color: var(--tx-3);
  margin-top: 6px;
}
.finish-foot {
  display: flex; justify-content: flex-end; gap: 10px;
  padding: 14px 22px;
  border-top: 1px solid var(--line-2);
  background: rgba(0,0,0,.25);
}
</style>
