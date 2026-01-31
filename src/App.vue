<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import Tesseract from 'tesseract.js'

const API_URL = 'https://api.tarkov.dev/graphql'
const PLAYER_API_URL = 'https://players.tarkov.dev/profile'
const STORAGE_KEY = 'tarkov_kappa_tracker_v2'
const CACHE_KEY = 'tarkov_kappa_task_cache'
const MAP_CACHE_KEY = 'tarkov_map_cache'
const SETTINGS_KEY = 'tarkov_tracker_settings'
const UI_STATE_KEY = 'tarkov_tracker_ui_state'

// Tarkov XP thresholds - cumulative XP needed for each level (index = level - 1)
const LEVEL_XP_THRESHOLDS = [
  0,          // Level 1
  1000,       // Level 2
  4017,       // Level 3
  8432,       // Level 4
  14256,      // Level 5
  21477,      // Level 6
  30023,      // Level 7
  39936,      // Level 8
  51204,      // Level 9
  63723,      // Level 10
  77563,      // Level 11
  93279,      // Level 12
  115302,     // Level 13
  143253,     // Level 14
  177337,     // Level 15
  217885,     // Level 16
  264432,     // Level 17
  316851,     // Level 18
  374400,     // Level 19
  437465,     // Level 20
  505161,     // Level 21
  577978,     // Level 22
  656347,     // Level 23
  741150,     // Level 24
  836066,     // Level 25
  944133,     // Level 26
  1066259,    // Level 27
  1199423,    // Level 28
  1343743,    // Level 29
  1499338,    // Level 30
  1666320,    // Level 31
  1846664,    // Level 32
  2043349,    // Level 33
  2258436,    // Level 34
  2492126,    // Level 35
  2750217,    // Level 36
  3032022,    // Level 37
  3337766,    // Level 38
  3663831,    // Level 39
  4010401,    // Level 40
  4377662,    // Level 41
  4765799,    // Level 42
  5182399,    // Level 43
  5627732,    // Level 44
  6102063,    // Level 45
  6630287,    // Level 46
  7189442,    // Level 47
  7779792,    // Level 48
  8401607,    // Level 49
  9055144,    // Level 50
  9740666,    // Level 51
  10458431,   // Level 52
  11219666,   // Level 53
  12024744,   // Level 54
  12874041,   // Level 55
  13767918,   // Level 56
  14706741,   // Level 57
  15690872,   // Level 58
  16720667,   // Level 59
  17816442,   // Level 60
  19041492,   // Level 61
  20360945,   // Level 62
  21792266,   // Level 63
  23350443,   // Level 64
  25098462,   // Level 65
  27100775,   // Level 66
  29581231,   // Level 67
  33028574,   // Level 68
  37953544,   // Level 69
  44260543,   // Level 70
  51901513,   // Level 71
  60887711,   // Level 72
  71228846,   // Level 73
  82933459,   // Level 74
  96009180,   // Level 75
  110462910,  // Level 76
  126300949,  // Level 77
  144924572,  // Level 78
  172016256   // Level 79
]

// Calculate player level from XP
function xpToLevel(xp) {
  for (let i = LEVEL_XP_THRESHOLDS.length - 1; i >= 0; i--) {
    if (xp >= LEVEL_XP_THRESHOLDS[i]) {
      return i + 1
    }
  }
  return 1
}

const query = `
  query {
    tasks {
      id
      name
      trader { name }
      map { name }
      minPlayerLevel
      experience
      wikiLink
      kappaRequired
      lightkeeperRequired
      taskRequirements {
        task { id name }
        status
      }
      objectives {
        id
        type
        description
        optional
        ... on TaskObjectiveItem {
          items { name shortName }
          count
          foundInRaid
        }
        ... on TaskObjectiveQuestItem {
          questItem { name }
          count
        }
        ... on TaskObjectiveMark {
          markerItem { name }
        }
      }
    }
  }
`

const allTasks = ref([])
const completedTasks = ref(new Set())
const currentFilter = ref('all')
const currentSort = ref({ column: null, direction: 'asc' })
const currentPage = ref('quests') // 'quests' or 'maps'
const currentView = ref('table') // 'table' or 'grid' (for quests page)
const searchTerm = ref('')
const loading = ref(true)
const error = ref('')
const syncing = ref(false)
const usingCache = ref(false)
const fabOpen = ref(false)

// Modal state
const showImportModal = ref(false)
const showSettingsModal = ref(false)

// Import modal state
const importDragOver = ref(false)
const importFiles = ref([])
const importProcessing = ref(false)
const importProgress = ref(0)
const importResults = ref([])
const ocrResults = ref([])

// Settings/Player state
const playerAccountId = ref('')
const playerData = ref(null)
const playerLevel = ref(null)
const playerLoadError = ref('')
const playerLoading = ref(false)
const levelFilterMode = ref('gray') // 'gray', 'hide', or 'show'

// Map page state
const selectedMap = ref(null)
const mapViewMode = ref('2d') // '2d', '3d', 'interactive'
const showMapInfo = ref(false)
const mapData = ref([]) // fetched from API

const mapQuery = `
  query {
    maps {
      id
      name
      normalizedName
      wiki
      description
      enemies
      raidDuration
      players
      minPlayerLevel
      maxPlayerLevel
      bosses { name spawnChance }
    }
  }
`

// Wiki interactive map URLs by normalizedName
const wikiMapUrls = {
  'customs': 'Customs',
  'factory': 'Factory',
  'ground-zero': 'Ground_Zero',
  'interchange': 'Interchange',
  'lighthouse': 'Lighthouse',
  'reserve': 'Reserve',
  'shoreline': 'Shoreline',
  'streets-of-tarkov': 'Streets_of_Tarkov',
  'the-lab': 'The_Lab',
  'woods': 'Woods',
}

// 2D map images by normalizedName (community maps from wiki)
const map2dUrls = {
  'customs': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/4/44/CustomsDormsByre3mr.jpg/revision/latest',
  'factory': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/3/33/Factory2DMapbyRe3mr.jpg/revision/latest',
  'ground-zero': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/9/9b/GroundZeroByxTycho.png/revision/latest',
  'interchange': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/2/27/InterchangeMapByre3mr.jpg/revision/latest',
  'lighthouse': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/1/13/Jindouz_Lighthouse_Map_V1.png/revision/latest',
  'reserve': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/7/7c/JindouzReserve_v1_2dmap.png/revision/latest',
  'shoreline': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/e/e5/Shoreline_2D_by_Jindouz.png/revision/latest',
  'streets-of-tarkov': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/c/c4/Streets_2D_by_Jindouz.png/revision/latest',
  'the-lab': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/b/bb/TheLabMapByMonkiAK.png/revision/latest',
  'woods': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/0/05/Glory4lyfeWoods_map_v4_marked.png/revision/latest',
}

// 3D map images by normalizedName
const map3dUrls = {
  'customs': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/0/0b/CustomsMapRealNorthColour_Reemr.png/revision/latest',
  'ground-zero': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/4/44/GroundZero3DMapByRe3mr.png/revision/latest',
  'interchange': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/2/27/InterchangeMapByre3mr.jpg/revision/latest',
  'lighthouse': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/9/9f/Re3mrLighthouseVerticalDay-min.jpg/revision/latest',
  'reserve': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/3/38/Re3mrReserveCardinalNorthWikiVer.jpg/revision/latest',
  'shoreline': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/2/25/Shoreline-3d-by-loweffort.png/revision/latest',
  'streets-of-tarkov': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/4/40/Streets-3d-by-loweffort.png/revision/latest',
  'the-lab': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/b/bb/TheLabMapByMonkiAK.png/revision/latest',
  'woods': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/d/de/WikiWoodsRe3mr.jpg/revision/latest',
}

const mapNames = computed(() => mapData.value.map(m => m.name).sort())

const selectedMapData = computed(() => {
  if (!selectedMap.value) return null
  return mapData.value.find(m => m.name === selectedMap.value) || null
})

const selectedMapNormalized = computed(() => selectedMapData.value?.normalizedName || '')

const interactiveMapUrl = computed(() => {
  const key = selectedMapNormalized.value
  const wikiName = wikiMapUrls[key]
  return wikiName ? `https://escapefromtarkov.fandom.com/wiki/Map:${wikiName}` : ''
})

const map2dUrl = computed(() => map2dUrls[selectedMapNormalized.value] || '')
const map3dUrl = computed(() => map3dUrls[selectedMapNormalized.value] || '')

// Tarkov time
const tarkovRatio = 7
function hrs(num) { return 1000 * 60 * 60 * num }

function realTimeToTarkovTime(time, left) {
  const oneDay = hrs(24)
  const russia = hrs(3)
  const offset = russia + (left ? 0 : hrs(12))
  return new Date((offset + (time.getTime() * tarkovRatio)) % oneDay)
}

function timeUntilRelative(until, left, date) {
  const tarkovTime = realTimeToTarkovTime(date, left)
  if (until < tarkovTime.getTime()) until += hrs(24)
  const diffTarkov = until - tarkovTime.getTime()
  return diffTarkov / tarkovRatio
}

function formatTarkovTime(date) {
  const h = String(date.getUTCHours()).padStart(2, '0')
  const m = String(date.getUTCMinutes()).padStart(2, '0')
  const s = String(date.getUTCSeconds()).padStart(2, '0')
  return `${h}:${m}:${s}`
}

function formatRealDuration(ms) {
  const totalSec = Math.floor(ms / 1000)
  const h = Math.floor(totalSec / 3600)
  const m = Math.floor((totalSec % 3600) / 60)
  const s = totalSec % 60
  if (h > 0) return `${h}h ${m}m ${s}s`
  if (m > 0) return `${m}m ${s}s`
  return `${s}s`
}

const tarkovTimeLeft = ref('')
const tarkovTimeRight = ref('')
const timeOffset = ref(0) // ms offset from scrolling
let timeInterval = null

function updateTarkovTime() {
  const now = new Date(Date.now() + timeOffset.value)
  tarkovTimeLeft.value = formatTarkovTime(realTimeToTarkovTime(now, true))
  tarkovTimeRight.value = formatTarkovTime(realTimeToTarkovTime(now, false))
}

function handleTimeScroll(e) {
  e.preventDefault()
  // Each scroll tick = 1 minute real time offset
  timeOffset.value += e.deltaY > 0 ? 60000 : -60000
  updateTarkovTime()
}

function resetTimeOffset() {
  timeOffset.value = 0
  updateTarkovTime()
}

// Computed: time until specific hour boundaries for display
const timeUntilDisplay = computed(() => {
  const now = new Date(Date.now() + timeOffset.value)
  // Common raid times: dawn (5:00), day (8:00), dusk (20:00), night (22:00)
  const points = [
    { label: 'Dawn', hour: 5 },
    { label: 'Day', hour: 8 },
    { label: 'Dusk', hour: 20 },
    { label: 'Night', hour: 22 },
  ]
  return points.map(p => {
    const targetMs = hrs(p.hour)
    const rtLeft = timeUntilRelative(targetMs, true, now)
    return { label: p.label, time: formatRealDuration(rtLeft) }
  })
})

// Build a map of taskId -> list of tasks that require it
const unlocksMap = computed(() => {
  const map = {}
  allTasks.value.forEach(task => {
    if (task.taskRequirements) {
      task.taskRequirements
        .filter(req => req.status && req.status.includes('complete'))
        .forEach(req => {
          if (!map[req.task.id]) map[req.task.id] = []
          map[req.task.id].push({ id: task.id, name: task.name })
        })
    }
  })
  return map
})

// Progress
const totalCount = computed(() => allTasks.value.length)
const completedCount = computed(() => completedTasks.value.size)
const progressPercent = computed(() =>
  totalCount.value > 0 ? Math.round((completedCount.value / totalCount.value) * 100) : 0
)
const kappaTotal = computed(() => allTasks.value.filter(t => t.kappaRequired).length)
const kappaCompleted = computed(() =>
  allTasks.value.filter(t => t.kappaRequired && completedTasks.value.has(t.id)).length
)

// Check if task is below player level
function isTaskBelowLevel(task) {
  if (!playerLevel.value) return false
  return task.minPlayerLevel > playerLevel.value
}

// Filtered & sorted tasks
const filteredTasks = computed(() => {
  let filtered = allTasks.value

  if (currentFilter.value === 'kappa') {
    filtered = filtered.filter(t => t.kappaRequired)
  } else if (currentFilter.value === 'incomplete') {
    filtered = filtered.filter(t => !completedTasks.value.has(t.id))
  } else if (currentFilter.value === 'kappa-todo') {
    filtered = filtered.filter(t => t.kappaRequired && !completedTasks.value.has(t.id))
  }

  // Hide tasks above player level if mode is 'hide'
  if (playerLevel.value && levelFilterMode.value === 'hide') {
    filtered = filtered.filter(t => t.minPlayerLevel <= playerLevel.value)
  }

  if (searchTerm.value) {
    const s = searchTerm.value.toLowerCase()
    filtered = filtered.filter(t =>
      t.name.toLowerCase().includes(s) ||
      t.trader.name.toLowerCase().includes(s) ||
      (t.map && t.map.name.toLowerCase().includes(s))
    )
  }

  if (currentSort.value.column) {
    filtered = sortTasks(filtered, currentSort.value.column, currentSort.value.direction)
  }

  return filtered
})

// Grid: tasks grouped by trader
const tasksByTrader = computed(() => {
  const groups = {}
  filteredTasks.value.forEach(task => {
    const trader = task.trader.name
    if (!groups[trader]) groups[trader] = []
    groups[trader].push(task)
  })
  return Object.keys(groups).sort().map(name => ({
    name,
    tasks: groups[name],
    completedCount: groups[name].filter(t => completedTasks.value.has(t.id)).length
  }))
})

function sortTasks(tasks, column, direction) {
  const sorted = [...tasks]
  sorted.sort((a, b) => {
    let aVal, bVal
    switch (column) {
      case 'trader': aVal = a.trader.name; bVal = b.trader.name; break
      case 'name': aVal = a.name; bVal = b.name; break
      case 'level': aVal = a.minPlayerLevel; bVal = b.minPlayerLevel; break
      case 'xp': aVal = a.experience; bVal = b.experience; break
      default: return 0
    }
    if (typeof aVal === 'string') {
      return direction === 'asc' ? aVal.localeCompare(bVal) : bVal.localeCompare(aVal)
    }
    return direction === 'asc' ? aVal - bVal : bVal - aVal
  })
  return sorted
}

function sortBy(column) {
  if (currentSort.value.column === column) {
    currentSort.value.direction = currentSort.value.direction === 'asc' ? 'desc' : 'asc'
  } else {
    currentSort.value.column = column
    currentSort.value.direction = 'asc'
  }
}

function sortClass(column) {
  if (currentSort.value.column !== column) return ''
  return currentSort.value.direction === 'asc' ? 'sorted-asc' : 'sorted-desc'
}

function setFilter(filter) {
  currentFilter.value = filter
}

function isCompleted(taskId) {
  return completedTasks.value.has(taskId)
}

function toggleTask(taskId) {
  const next = new Set(completedTasks.value)
  if (next.has(taskId)) {
    next.delete(taskId)
  } else {
    next.add(taskId)
  }
  completedTasks.value = next
  saveProgress()
}

function getPrerequisites(requirements) {
  if (!requirements || requirements.length === 0) return []
  return requirements
    .filter(req => req.status && req.status.includes('complete'))
    .map(req => ({
      name: req.task.name,
      completed: completedTasks.value.has(req.task.id)
    }))
}

function getUnlocks(taskId) {
  return unlocksMap.value[taskId] || []
}

function getObjectives(objectives) {
  if (!objectives || objectives.length === 0) return []
  return objectives.map(obj => ({
    text: obj.description,
    shortText: obj.description.length > 80 ? obj.description.substring(0, 80) + '...' : obj.description,
    optional: obj.optional
  }))
}

function getItems(objectives) {
  if (!objectives || objectives.length === 0) return []
  const items = []
  objectives.forEach(obj => {
    if (obj.items && obj.items.length > 0) {
      obj.items.forEach(item => {
        items.push({
          name: item.shortName || item.name,
          count: obj.count > 1 ? obj.count : null,
          fir: obj.foundInRaid
        })
      })
    }
    if (obj.questItem) {
      items.push({
        name: obj.questItem.name,
        count: obj.count > 1 ? obj.count : null,
        fir: false
      })
    }
    if (obj.markerItem) {
      items.push({ name: obj.markerItem.name, count: null, fir: false })
    }
  })
  return items
}

function toggleView() {
  currentView.value = currentView.value === 'table' ? 'grid' : 'table'
}

function saveProgress() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify([...completedTasks.value]))
}

function loadProgress() {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    completedTasks.value = new Set(JSON.parse(saved))
  }
}

function saveCache(tasks) {
  localStorage.setItem(CACHE_KEY, JSON.stringify({ tasks, timestamp: Date.now() }))
}

function loadCache() {
  const saved = localStorage.getItem(CACHE_KEY)
  if (saved) {
    try { return JSON.parse(saved) } catch { return null }
  }
  return null
}

function openWiki(url) {
  window.open(url, '_blank')
}

function exportData() {
  const data = {
    version: 2,
    completed: [...completedTasks.value],
    timestamp: new Date().toISOString()
  }
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `kappa-tracker-${Date.now()}.json`
  a.click()
  URL.revokeObjectURL(url)
}

function importData() {
  // Open import modal instead of simple file picker
  showImportModal.value = true
}

// Close import modal and reset state
function closeImportModal() {
  showImportModal.value = false
  importDragOver.value = false
  importFiles.value = []
  importProcessing.value = false
  importProgress.value = 0
  importResults.value = []
  ocrResults.value = []
}

// Handle drag over for import modal
function handleImportDragOver(e) {
  e.preventDefault()
  importDragOver.value = true
}

function handleImportDragLeave(e) {
  e.preventDefault()
  importDragOver.value = false
}

// Handle file drop in import modal
function handleImportDrop(e) {
  e.preventDefault()
  importDragOver.value = false
  const files = Array.from(e.dataTransfer.files)
  addImportFiles(files)
}

// Handle file selection from file input
function handleImportFileSelect(e) {
  const files = Array.from(e.target.files)
  addImportFiles(files)
  e.target.value = '' // Reset input
}

// Add files to import list
function addImportFiles(files) {
  files.forEach(file => {
    const isJson = file.type === 'application/json' || file.name.endsWith('.json')
    const isImage = file.type.startsWith('image/')
    if (isJson || isImage) {
      importFiles.value.push({
        file,
        type: isJson ? 'json' : 'image',
        name: file.name,
        status: 'pending',
        preview: isImage ? URL.createObjectURL(file) : null
      })
    }
  })
}

// Remove file from import list
function removeImportFile(index) {
  const file = importFiles.value[index]
  if (file.preview) URL.revokeObjectURL(file.preview)
  importFiles.value.splice(index, 1)
}

// Process all import files
async function processImportFiles() {
  if (importFiles.value.length === 0) return

  importProcessing.value = true
  importProgress.value = 0
  importResults.value = []
  ocrResults.value = []

  const total = importFiles.value.length
  let processed = 0

  for (const item of importFiles.value) {
    item.status = 'processing'

    try {
      if (item.type === 'json') {
        await processJsonFile(item)
      } else {
        await processImageFile(item)
      }
      item.status = 'done'
    } catch (err) {
      item.status = 'error'
      item.error = err.message
    }

    processed++
    importProgress.value = Math.round((processed / total) * 100)
  }

  // Apply OCR results if any
  if (ocrResults.value.length > 0) {
    applyOcrResults()
  }

  importProcessing.value = false
}

// Process JSON file
async function processJsonFile(item) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = (event) => {
      try {
        const data = JSON.parse(event.target.result)
        if (data.completed && Array.isArray(data.completed)) {
          data.completed.forEach(id => completedTasks.value.add(id))
          saveProgress()
          importResults.value.push({
            type: 'json',
            file: item.name,
            count: data.completed.length,
            message: `Imported ${data.completed.length} completed tasks`
          })
        }
        resolve()
      } catch (err) {
        reject(err)
      }
    }
    reader.onerror = () => reject(new Error('Failed to read file'))
    reader.readAsText(item.file)
  })
}

// Process image file with OCR
async function processImageFile(item) {
  const result = await Tesseract.recognize(item.file, 'eng', {
    logger: m => {
      if (m.status === 'recognizing text') {
        item.ocrProgress = Math.round(m.progress * 100)
      }
    }
  })

  const text = result.data.text
  const parsed = parseOcrText(text)

  ocrResults.value.push({
    file: item.name,
    raw: text,
    parsed
  })

  importResults.value.push({
    type: 'ocr',
    file: item.name,
    count: parsed.length,
    message: `Found ${parsed.length} quests in screenshot`
  })
}

// Parse OCR text to extract quest names and statuses
function parseOcrText(text) {
  const lines = text.split('\n').map(l => l.trim()).filter(l => l.length > 3)
  const quests = []

  for (const line of lines) {
    // Check for status indicators
    const isCompleted = /completed|complete/i.test(line)
    const isActive = /active!?|active\s*!/i.test(line)
    const isLocked = /locked|\u{1F512}|🔒/ui.test(line) || line.includes('🔒')

    // Extract quest name by removing status text
    let questName = line
      .replace(/completed|complete|active!?|active\s*!|locked/gi, '')
      .replace(/[^\w\s\-'":]/g, '') // Remove special chars but keep apostrophes, quotes, colons
      .trim()

    if (questName.length < 3) continue

    // Determine status
    let status = 'unknown'
    if (isCompleted) status = 'completed'
    else if (isActive) status = 'active'
    else if (isLocked) status = 'locked'

    quests.push({ name: questName, status, originalLine: line })
  }

  return quests
}

// Fuzzy match quest name to actual task
function fuzzyMatchQuest(name) {
  const normalizedName = name.toLowerCase().replace(/[^a-z0-9]/g, '')
  let bestMatch = null
  let bestScore = 0

  for (const task of allTasks.value) {
    const taskNormalized = task.name.toLowerCase().replace(/[^a-z0-9]/g, '')

    // Exact match
    if (taskNormalized === normalizedName) {
      return { task, score: 1 }
    }

    // Contains match
    if (taskNormalized.includes(normalizedName) || normalizedName.includes(taskNormalized)) {
      const score = Math.min(normalizedName.length, taskNormalized.length) /
                   Math.max(normalizedName.length, taskNormalized.length)
      if (score > bestScore) {
        bestScore = score
        bestMatch = task
      }
    }

    // Levenshtein-like similarity for fuzzy matching
    const similarity = calculateSimilarity(normalizedName, taskNormalized)
    if (similarity > bestScore && similarity > 0.6) {
      bestScore = similarity
      bestMatch = task
    }
  }

  return bestMatch ? { task: bestMatch, score: bestScore } : null
}

// Simple similarity calculation
function calculateSimilarity(str1, str2) {
  if (str1 === str2) return 1
  if (str1.length === 0 || str2.length === 0) return 0

  // Count matching characters
  let matches = 0
  const longer = str1.length > str2.length ? str1 : str2
  const shorter = str1.length > str2.length ? str2 : str1

  for (let i = 0; i < shorter.length; i++) {
    if (longer.includes(shorter[i])) matches++
  }

  // Check for common substrings
  let commonLength = 0
  for (let len = Math.min(str1.length, str2.length); len >= 3; len--) {
    for (let i = 0; i <= str1.length - len; i++) {
      const substr = str1.substring(i, i + len)
      if (str2.includes(substr)) {
        commonLength = Math.max(commonLength, len)
        break
      }
    }
    if (commonLength > 0) break
  }

  const charSimilarity = matches / longer.length
  const substrSimilarity = commonLength / Math.max(str1.length, str2.length)

  return Math.max(charSimilarity * 0.5 + substrSimilarity * 0.5, substrSimilarity)
}

// Apply OCR results to mark quests
function applyOcrResults() {
  let marked = 0
  const matched = []

  for (const result of ocrResults.value) {
    for (const quest of result.parsed) {
      const match = fuzzyMatchQuest(quest.name)
      if (match && match.score > 0.5) {
        matched.push({
          ocrName: quest.name,
          taskName: match.task.name,
          status: quest.status,
          score: match.score
        })

        // Mark completed or active quests
        if (quest.status === 'completed' || quest.status === 'active') {
          if (!completedTasks.value.has(match.task.id)) {
            // For active quests, mark prerequisites as completed
            if (quest.status === 'active') {
              markPrerequisitesCompleted(match.task)
            }

            // Mark the quest itself as completed if status is completed
            if (quest.status === 'completed') {
              completedTasks.value.add(match.task.id)
              marked++
            }
          }
        }
      }
    }
  }

  if (marked > 0) {
    saveProgress()
  }

  importResults.value.push({
    type: 'summary',
    message: `Matched ${matched.length} quests, marked ${marked} as completed`
  })
}

// Mark all prerequisites of a task as completed
function markPrerequisitesCompleted(task) {
  if (!task.taskRequirements) return

  for (const req of task.taskRequirements) {
    if (req.status && req.status.includes('complete') && req.task) {
      if (!completedTasks.value.has(req.task.id)) {
        completedTasks.value.add(req.task.id)
        // Recursively mark prerequisites
        const prereqTask = allTasks.value.find(t => t.id === req.task.id)
        if (prereqTask) {
          markPrerequisitesCompleted(prereqTask)
        }
      }
    }
  }
}

// Settings functions
function saveSettings() {
  const settings = {
    playerAccountId: playerAccountId.value,
    levelFilterMode: levelFilterMode.value
  }
  localStorage.setItem(SETTINGS_KEY, JSON.stringify(settings))
}

function loadSettings() {
  const saved = localStorage.getItem(SETTINGS_KEY)
  if (saved) {
    try {
      const settings = JSON.parse(saved)
      playerAccountId.value = settings.playerAccountId || ''
      levelFilterMode.value = settings.levelFilterMode || 'gray'
    } catch { /* ignore */ }
  }
}

// UI state persistence
function saveUiState() {
  const state = {
    currentFilter: currentFilter.value,
    currentView: currentView.value,
    currentPage: currentPage.value,
    currentSort: currentSort.value,
    selectedMap: selectedMap.value,
    mapViewMode: mapViewMode.value
  }
  localStorage.setItem(UI_STATE_KEY, JSON.stringify(state))
}

function loadUiState() {
  const saved = localStorage.getItem(UI_STATE_KEY)
  if (saved) {
    try {
      const state = JSON.parse(saved)
      currentFilter.value = state.currentFilter || 'all'
      currentView.value = state.currentView || 'table'
      currentPage.value = state.currentPage || 'quests'
      currentSort.value = state.currentSort || { column: null, direction: 'asc' }
      selectedMap.value = state.selectedMap || null
      mapViewMode.value = state.mapViewMode || '2d'
    } catch { /* ignore */ }
  }
}

// Watch for UI state changes and save
watch([currentFilter, currentView, currentPage, currentSort, selectedMap, mapViewMode], () => {
  saveUiState()
}, { deep: true })

// Fetch player data
async function fetchPlayerData() {
  if (!playerAccountId.value) {
    playerData.value = null
    playerLevel.value = null
    return
  }

  playerLoading.value = true
  playerLoadError.value = ''

  try {
    const response = await fetch(
  `${PLAYER_API_URL}/${playerAccountId.value}.json`
)
    if (!response.ok) {
      throw new Error('Player not found')
    }
    const data = await response.json()
    playerData.value = data
    playerLevel.value = xpToLevel(data.info.experience)
    saveSettings()
  } catch (err) {
    playerLoadError.value = err.message
    playerData.value = null
    playerLevel.value = null
  } finally {
    playerLoading.value = false
  }
}

// Open settings modal
function openSettings() {
  showSettingsModal.value = true
}

// Close settings modal
function closeSettingsModal() {
  showSettingsModal.value = false
}

// Save settings and close modal
function saveAndCloseSettings() {
  saveSettings()
  closeSettingsModal()
}

async function fetchMaps() {
  const response = await fetch(API_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query: mapQuery })
  })
  const result = await response.json()
  if (result.errors) throw new Error(result.errors[0].message)
  return result.data.maps
}

async function fetchFromApi() {
  const response = await fetch(API_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query })
  })
  const result = await response.json()
  if (result.errors) throw new Error(result.errors[0].message)
  return result.data.tasks
}

onMounted(async () => {
  loadProgress()
  loadSettings()
  loadUiState()

  // Start tarkov time ticker
  updateTarkovTime()
  timeInterval = setInterval(updateTarkovTime, 1000)

  // Try loading from cache first
  const cached = loadCache()
  if (cached && cached.tasks && cached.tasks.length > 0) {
    allTasks.value = cached.tasks
    usingCache.value = true
    loading.value = false
  }
  try {
    const cachedMaps = localStorage.getItem(MAP_CACHE_KEY)
    if (cachedMaps) mapData.value = JSON.parse(cachedMaps)
  } catch { /* ignore */ }

  // Fetch fresh data from API
  syncing.value = true
  try {
    const [tasks, maps] = await Promise.all([fetchFromApi(), fetchMaps()])
    allTasks.value = tasks
    mapData.value = maps
    saveCache(tasks)
    localStorage.setItem(MAP_CACHE_KEY, JSON.stringify(maps))
    usingCache.value = false
    syncing.value = false
    loading.value = false

    // Fetch player data after levels are loaded (so we can calculate level from XP)
    if (playerAccountId.value) {
      fetchPlayerData()
    }
  } catch (err) {
    syncing.value = false
    if (allTasks.value.length === 0) {
      loading.value = false
      error.value = err.message
    }
    usingCache.value = allTasks.value.length > 0
  }
})

onUnmounted(() => {
  if (timeInterval) clearInterval(timeInterval)
})
</script>

<template>
  <!-- Quests Page -->
  <template v-if="currentPage === 'quests'">
    <!-- Progress Bar Header -->
    <div class="progress-bar-container">
      <div class="progress-info">
        <div>
          <span class="progress-label">Progress:</span>
          <span class="progress-value">{{ completedCount }} / {{ totalCount }}</span>
        </div>
        <div>
          <span class="progress-label">Kappa:</span>
          <span class="progress-value">{{ kappaCompleted }} / {{ kappaTotal }}</span>
        </div>
        <div v-if="syncing" class="sync-indicator syncing">SYNCING...</div>
        <div v-else-if="usingCache" class="sync-indicator cached">CACHED</div>
        <div v-else-if="!loading && !error" class="sync-indicator synced">SYNCED</div>
      </div>

      <div class="progress-bar">
        <div class="progress-fill" :style="{ width: progressPercent + '%' }"></div>
      </div>

      <div class="controls">
        <input
          type="text"
          class="search-input"
          v-model="searchTerm"
          placeholder="Search tasks..."
        />
        <button
          class="filter-btn"
          :class="{ active: currentFilter === 'all' }"
          @click="setFilter('all')"
        >ALL</button>
        <button
          class="filter-btn"
          :class="{ active: currentFilter === 'kappa' }"
          @click="setFilter('kappa')"
        >KAPPA</button>
        <button
          class="filter-btn"
          :class="{ active: currentFilter === 'incomplete' }"
          @click="setFilter('incomplete')"
        >TODO</button>
        <button
          class="filter-btn"
          :class="{ active: currentFilter === 'kappa-todo' }"
          @click="setFilter('kappa-todo')"
        >KAPPA TODO</button>
        <button class="export-btn" @click="toggleView">
          {{ currentView === 'table' ? 'GRID' : 'TABLE' }}
        </button>
        <button class="export-btn" @click="exportData">EXPORT</button>
        <button class="export-btn" @click="importData">IMPORT</button>
      </div>
    </div>

    <!-- Table View -->
    <div v-if="currentView === 'table'" class="table-container">
      <table>
        <thead>
          <tr>
            <th class="checkbox-cell">✓</th>
            <th class="trader-cell sortable" :class="sortClass('trader')" @click="sortBy('trader')">TRADER</th>
            <th class="task-name-cell sortable" :class="sortClass('name')" @click="sortBy('name')">TASK NAME</th>
            <th class="level-cell sortable" :class="sortClass('level')" @click="sortBy('level')">LVL</th>
            <th class="prereq-cell">REQUIRES</th>
            <th class="unlocks-cell">UNLOCKS</th>
            <th class="map-cell">MAP</th>
            <th class="xp-cell sortable" :class="sortClass('xp')" @click="sortBy('xp')">XP</th>
            <th class="objectives-cell">OBJECTIVES</th>
            <th class="items-cell">REQUIRED ITEMS</th>
            <th class="badges-cell">TAGS</th>
            <th class="wiki-cell">LINK</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="task in filteredTasks"
            :key="task.id"
            :class="{
              completed: isCompleted(task.id),
              'below-level': levelFilterMode === 'gray' && isTaskBelowLevel(task)
            }"
          >
            <td class="checkbox-cell">
              <div
                class="task-checkbox"
                :class="{ checked: isCompleted(task.id) }"
                @click="toggleTask(task.id)"
              ></div>
            </td>
            <td class="trader-cell">{{ task.trader.name }}</td>
            <td class="task-name-cell" @dblclick="openWiki(task.wikiLink)">{{ task.name }}</td>
            <td class="level-cell">{{ task.minPlayerLevel }}</td>
            <td class="prereq-cell">
              <template v-if="getPrerequisites(task.taskRequirements).length">
                <span
                  v-for="(req, i) in getPrerequisites(task.taskRequirements)"
                  :key="i"
                  class="prereq-item"
                  :class="req.completed ? 'completed' : 'incomplete'"
                >{{ req.name }}</span>
              </template>
              <template v-else>-</template>
            </td>
            <td class="unlocks-cell">
              <template v-if="getUnlocks(task.id).length">
                <div class="hover-count-wrapper">
                  <span class="hover-count">{{ getUnlocks(task.id).length }}</span>
                  <div class="hover-list">
                    <div
                      v-for="(u, i) in getUnlocks(task.id)"
                      :key="i"
                      class="hover-list-item"
                      :class="{ 'is-completed': isCompleted(u.id) }"
                    >{{ u.name }}</div>
                  </div>
                </div>
              </template>
              <template v-else>-</template>
            </td>
            <td class="map-cell">{{ task.map ? task.map.name : '-' }}</td>
            <td class="xp-cell">{{ task.experience.toLocaleString() }}</td>
            <td class="objectives-cell">
              <template v-if="getObjectives(task.objectives).length">
                <div class="hover-count-wrapper">
                  <span class="hover-count">{{ getObjectives(task.objectives).length }}</span>
                  <div class="hover-list">
                    <div
                      v-for="(obj, i) in getObjectives(task.objectives)"
                      :key="i"
                      class="hover-list-item"
                      :class="{ 'is-optional': obj.optional }"
                    >{{ obj.text }}</div>
                  </div>
                </div>
              </template>
              <template v-else>-</template>
            </td>
            <td class="items-cell">
              <template v-if="getItems(task.objectives).length">
                <div class="hover-count-wrapper">
                  <span class="hover-count">{{ getItems(task.objectives).length }}</span>
                  <div class="hover-list">
                    <div
                      v-for="(item, i) in getItems(task.objectives)"
                      :key="i"
                      class="hover-list-item"
                      :class="{ 'is-fir': item.fir }"
                    >{{ item.name }}{{ item.count ? ` x${item.count}` : '' }}{{ item.fir ? ' (FIR)' : '' }}</div>
                  </div>
                </div>
              </template>
              <template v-else>-</template>
            </td>
            <td class="badges-cell">
              <span v-if="task.kappaRequired" class="badge kappa">K</span>
              <span v-if="task.lightkeeperRequired" class="badge lightkeeper">LK</span>
            </td>
            <td class="wiki-cell">
              <a :href="task.wikiLink" target="_blank" class="wiki-link">WIKI</a>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Grid View -->
    <div v-else class="grid-view active">
      <div v-for="group in tasksByTrader" :key="group.name" class="trader-grid-section">
        <div class="trader-grid-header">
          {{ group.name }}
          <div class="trader-grid-stats">{{ group.completedCount }} / {{ group.tasks.length }} complete</div>
        </div>
        <div
          v-for="task in group.tasks"
          :key="task.id"
          class="grid-task-item"
          :class="{
            completed: isCompleted(task.id),
            'below-level': levelFilterMode === 'gray' && isTaskBelowLevel(task)
          }"
        >
          <div
            class="grid-task-checkbox"
            :class="{ checked: isCompleted(task.id) }"
            @click="toggleTask(task.id)"
          ></div>
          <div class="grid-task-name" @dblclick="openWiki(task.wikiLink)">
            {{ task.name }}
          </div>
          <div class="grid-task-badges">
            <span v-if="task.kappaRequired" class="badge kappa grid-badge">K</span>
            <span v-if="task.lightkeeperRequired" class="badge lightkeeper grid-badge">LK</span>
          </div>
        </div>
      </div>
    </div>
  </template>

  <!-- Maps Page -->
  <template v-if="currentPage === 'maps'">
    <div class="maps-page">
      <div class="maps-header">
        <div class="maps-tab-bar">
          <button
            v-for="name in mapNames"
            :key="name"
            class="map-tab"
            :class="{ active: selectedMap === name }"
            @click="selectedMap = name; showMapInfo = false"
          >{{ name }}</button>
        </div>
        <div class="maps-header-right">
          <button
            v-if="selectedMap"
            class="map-info-btn"
            :class="{ active: showMapInfo }"
            @click="showMapInfo = !showMapInfo"
            title="Map Info"
          >INFO</button>
          <div class="map-view-toggle">
            <button
              class="map-view-btn"
              :class="{ active: mapViewMode === '2d' }"
              @click="mapViewMode = '2d'"
            >2D</button>
            <button
              class="map-view-btn"
              :class="{ active: mapViewMode === '3d' }"
              @click="mapViewMode = '3d'"
            >3D</button>
            <button
              class="map-view-btn"
              :class="{ active: mapViewMode === 'interactive' }"
              @click="mapViewMode = 'interactive'"
            >INTERACTIVE</button>
          </div>
        </div>
      </div>
      <div class="maps-content">
        <div v-if="!selectedMap" class="maps-placeholder">
          SELECT A MAP
        </div>
        <template v-else>
          <!-- Info Panel -->
          <div v-if="showMapInfo && selectedMapData" class="map-info-panel">
            <div class="map-info-title">{{ selectedMapData.name }}</div>
            <div v-if="selectedMapData.description" class="map-info-desc">{{ selectedMapData.description }}</div>
            <div class="map-info-grid">
              <div class="map-info-item">
                <span class="map-info-label">RAID DURATION</span>
                <span class="map-info-value">{{ selectedMapData.raidDuration }}min</span>
              </div>
              <div class="map-info-item">
                <span class="map-info-label">PLAYERS</span>
                <span class="map-info-value">{{ selectedMapData.players }}</span>
              </div>
              <div v-if="selectedMapData.minPlayerLevel" class="map-info-item">
                <span class="map-info-label">MIN LEVEL</span>
                <span class="map-info-value">{{ selectedMapData.minPlayerLevel }}</span>
              </div>
              <div v-if="selectedMapData.maxPlayerLevel" class="map-info-item">
                <span class="map-info-label">MAX LEVEL</span>
                <span class="map-info-value">{{ selectedMapData.maxPlayerLevel }}</span>
              </div>
            </div>
            <div v-if="selectedMapData.enemies && selectedMapData.enemies.length" class="map-info-section">
              <div class="map-info-section-title">ENEMIES</div>
              <div class="map-info-tags">
                <span v-for="e in selectedMapData.enemies" :key="e" class="map-info-tag">{{ e }}</span>
              </div>
            </div>
            <div v-if="selectedMapData.bosses && selectedMapData.bosses.length" class="map-info-section">
              <div class="map-info-section-title">BOSSES</div>
              <div v-for="b in selectedMapData.bosses" :key="b.name" class="map-info-boss">
                <span class="boss-name">{{ b.name }}</span>
                <span class="boss-chance">{{ Math.round(b.spawnChance * 100) }}%</span>
              </div>
            </div>
          </div>
          <!-- Map View -->
          <iframe
            v-if="mapViewMode === 'interactive' && interactiveMapUrl"
            :src="interactiveMapUrl"
            class="map-iframe"
            allowfullscreen
          ></iframe>
          <div v-else-if="mapViewMode === '2d' && map2dUrl" class="map-image-container">
            <img :src="map2dUrl" :alt="selectedMap + ' 2D Map'" class="map-image" />
          </div>
          <div v-else-if="mapViewMode === '3d' && map3dUrl" class="map-image-container">
            <img :src="map3dUrl" :alt="selectedMap + ' 3D Map'" class="map-image" />
          </div>
          <div v-else class="maps-placeholder">
            NO MAP AVAILABLE
          </div>
        </template>
      </div>
    </div>
  </template>

  <!-- Loading & Error -->
  <div v-if="loading" id="loading">LOADING TASKS...</div>
  <div v-if="error" id="error">ERROR: {{ error }}</div>

  <!-- Tarkov Time Widget (bottom left) -->
  <div class="tarkov-time-widget" @wheel="handleTimeScroll">
    <div class="tarkov-time-header">
      TARKOV TIME
      <span v-if="timeOffset !== 0" class="time-reset" @click="resetTimeOffset">RESET</span>
    </div>
    <div class="tarkov-time-row">
      <span class="time-label">L:</span>
      <span class="time-value">{{ tarkovTimeLeft }}</span>
    </div>
    <div class="tarkov-time-row">
      <span class="time-label">R:</span>
      <span class="time-value">{{ tarkovTimeRight }}</span>
    </div>
    <div v-if="timeOffset !== 0" class="tarkov-time-offsets">
      <div v-for="point in timeUntilDisplay" :key="point.label" class="time-until-item">
        <span class="time-until-label">{{ point.label }}:</span>
        <span class="time-until-value">{{ point.time }}</span>
      </div>
    </div>
    <div v-if="timeOffset !== 0" class="time-offset-hint">
      {{ timeOffset > 0 ? '+' : '' }}{{ Math.round(timeOffset / 60000) }}m offset
    </div>
    <div v-else class="time-offset-hint">scroll to advance</div>
  </div>

  <!-- FAB Navigation (bottom right) -->
  <div class="fab-container">
    <transition name="fab-child">
      <div v-if="fabOpen" class="fab-children">
        <button
          class="fab-child-btn"
          :class="{ active: currentPage === 'quests' }"
          @click="currentPage = 'quests'; fabOpen = false"
          title="Quests"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" width="22" height="22">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
            <polyline points="14 2 14 8 20 8"/>
            <line x1="16" y1="13" x2="8" y2="13"/>
            <line x1="16" y1="17" x2="8" y2="17"/>
            <polyline points="10 9 9 9 8 9"/>
          </svg>
        </button>
        <button
          class="fab-child-btn"
          :class="{ active: currentPage === 'maps' }"
          @click="currentPage = 'maps'; fabOpen = false"
          title="Maps"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" width="22" height="22">
            <polygon points="1 6 1 22 8 18 16 22 23 18 23 2 16 6 8 2 1 6"/>
            <line x1="8" y1="2" x2="8" y2="18"/>
            <line x1="16" y1="6" x2="16" y2="22"/>
          </svg>
        </button>
        <button
          class="fab-child-btn"
          @click="openSettings(); fabOpen = false"
          title="Settings"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" width="22" height="22">
            <circle cx="12" cy="12" r="3"/>
            <path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"/>
          </svg>
        </button>
      </div>
    </transition>
    <button class="fab-button" @click="fabOpen = !fabOpen">
      <span class="fab-icon" :class="{ open: fabOpen }">+</span>
    </button>
  </div>

  <!-- Import Modal -->
  <div v-if="showImportModal" class="modal-overlay" @click.self="closeImportModal">
    <div class="modal-content import-modal">
      <div class="modal-header">
        <h2>Import Progress</h2>
        <button class="modal-close" @click="closeImportModal">&times;</button>
      </div>

      <div class="modal-body">
        <!-- Drop Zone -->
        <div
          class="drop-zone"
          :class="{ 'drag-over': importDragOver, 'has-files': importFiles.length > 0 }"
          @dragover="handleImportDragOver"
          @dragleave="handleImportDragLeave"
          @drop="handleImportDrop"
        >
          <div v-if="importFiles.length === 0" class="drop-zone-content">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="48" height="48">
              <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
              <polyline points="17 8 12 3 7 8"/>
              <line x1="12" y1="3" x2="12" y2="15"/>
            </svg>
            <p>Drag & drop files here</p>
            <p class="drop-zone-hint">JSON files or screenshots of your quest list</p>
          </div>

          <!-- File List -->
          <div v-else class="file-list">
            <div v-for="(item, index) in importFiles" :key="index" class="file-item">
              <img v-if="item.preview" :src="item.preview" class="file-preview" alt="Preview" />
              <div v-else class="file-icon">JSON</div>
              <div class="file-info">
                <div class="file-name">{{ item.name }}</div>
                <div class="file-status" :class="item.status">
                  {{ item.status === 'pending' ? 'Ready' : item.status === 'processing' ? 'Processing...' : item.status === 'done' ? 'Done' : 'Error' }}
                  <span v-if="item.ocrProgress && item.status === 'processing'">({{ item.ocrProgress }}%)</span>
                </div>
              </div>
              <button v-if="!importProcessing" class="file-remove" @click="removeImportFile(index)">&times;</button>
            </div>
          </div>
        </div>

        <!-- File Input -->
        <div class="import-actions">
          <label class="import-btn">
            <input type="file" multiple accept=".json,image/*" @change="handleImportFileSelect" hidden />
            Browse Files
          </label>
          <button
            v-if="importFiles.length > 0"
            class="import-btn primary"
            :disabled="importProcessing"
            @click="processImportFiles"
          >
            {{ importProcessing ? `Processing... ${importProgress}%` : 'Import' }}
          </button>
        </div>

        <!-- Progress Bar -->
        <div v-if="importProcessing" class="import-progress">
          <div class="import-progress-bar" :style="{ width: importProgress + '%' }"></div>
        </div>

        <!-- Results -->
        <div v-if="importResults.length > 0" class="import-results">
          <div class="results-header">Results</div>
          <div v-for="(result, index) in importResults" :key="index" class="result-item">
            <span class="result-type">{{ result.type.toUpperCase() }}</span>
            <span class="result-message">{{ result.message }}</span>
          </div>
        </div>

        <!-- Example Screenshot -->
        <div class="import-example">
          <div class="example-header">Example Screenshot Format</div>
          <div class="example-content">
            <p>Upload screenshots showing your quest list with statuses:</p>
            <ul>
              <li><span class="status-completed">completed</span> - Quest will be marked as done</li>
              <li><span class="status-active">active!</span> - Prerequisites will be marked as done</li>
              <li><span class="status-locked">locked</span> - Quest will be skipped</li>
            </ul>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Settings Modal -->
  <div v-if="showSettingsModal" class="modal-overlay" @click.self="closeSettingsModal">
    <div class="modal-content settings-modal">
      <div class="modal-header">
        <h2>Settings</h2>
        <button class="modal-close" @click="closeSettingsModal">&times;</button>
      </div>

      <div class="modal-body">
        <!-- Player Account Section -->
        <div class="settings-section">
          <div class="settings-section-title">Player Profile</div>

          <div class="settings-field">
            <label>Account ID</label>
            <div class="input-row">
              <input
                type="text"
                v-model="playerAccountId"
                placeholder="e.g., 5362033"
                class="settings-input"
              />
              <button class="settings-btn" @click="fetchPlayerData" :disabled="playerLoading">
                {{ playerLoading ? 'Loading...' : 'Load' }}
              </button>
            </div>
            <div class="settings-hint">
              Find your account ID at <a href="https://tarkov.dev/players" target="_blank">tarkov.dev/players</a>
            </div>
          </div>

          <div v-if="playerLoadError" class="settings-error">
            {{ playerLoadError }}
          </div>

          <div v-if="playerData" class="player-info-card">
            <div class="player-info-row">
              <span class="player-label">Nickname:</span>
              <span class="player-value">{{ playerData.info.nickname }}</span>
            </div>
            <div class="player-info-row">
              <span class="player-label">Level:</span>
              <span class="player-value player-level">{{ playerLevel }}</span>
            </div>
            <div class="player-info-row">
              <span class="player-label">Side:</span>
              <span class="player-value">{{ playerData.info.side }}</span>
            </div>
            <div class="player-info-row">
              <span class="player-label">Experience:</span>
              <span class="player-value">{{ playerData.info.experience.toLocaleString() }}</span>
            </div>
          </div>
        </div>

        <!-- Level Filter Section -->
        <div class="settings-section">
          <div class="settings-section-title">Quest Level Filter</div>
          <div class="settings-hint">How to display quests above your current level</div>

          <div class="radio-group">
            <label class="radio-option" :class="{ active: levelFilterMode === 'gray' }">
              <input type="radio" v-model="levelFilterMode" value="gray" />
              <span class="radio-label">Gray out</span>
              <span class="radio-desc">Show quests but visually muted</span>
            </label>
            <label class="radio-option" :class="{ active: levelFilterMode === 'hide' }">
              <input type="radio" v-model="levelFilterMode" value="hide" />
              <span class="radio-label">Hide</span>
              <span class="radio-desc">Don't show quests until you reach the level</span>
            </label>
            <label class="radio-option" :class="{ active: levelFilterMode === 'show' }">
              <input type="radio" v-model="levelFilterMode" value="show" />
              <span class="radio-label">Show all</span>
              <span class="radio-desc">Show all quests regardless of level</span>
            </label>
          </div>
        </div>
      </div>

      <div class="modal-footer">
        <button class="settings-btn" @click="closeSettingsModal">Cancel</button>
        <button class="settings-btn primary" @click="saveAndCloseSettings">Save Settings</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Progress Bar at Top */
.progress-bar-container {
    position: sticky;
    top: 0;
    z-index: 1000;
    background: var(--bg-secondary);
    border-bottom: 2px solid var(--border);
    padding: 8px 20px;
    display: flex;
    align-items: center;
    gap: 20px;
    box-shadow: 0 2px 10px var(--shadow);
}

.progress-info {
    display: flex;
    align-items: center;
    gap: 15px;
    flex-wrap: wrap;
}

.progress-label {
    font-size: 0.75rem;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--text-muted);
}

.progress-value {
    font-size: 0.9rem;
    font-weight: 700;
    color: var(--accent-primary);
}

/* Sync Indicator */
.sync-indicator {
    font-size: 0.65rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    padding: 2px 8px;
    border: 1px solid;
}

.sync-indicator.syncing {
    color: var(--kappa);
    border-color: var(--kappa);
    animation: pulse 1.5s ease-in-out infinite;
}

.sync-indicator.cached {
    color: var(--text-muted);
    border-color: var(--text-muted);
}

.sync-indicator.synced {
    color: var(--success);
    border-color: var(--success);
}

@keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.4; }
}

.progress-bar {
    flex: 1;
    height: 12px;
    background: var(--bg-primary);
    border: 1px solid var(--border);
    position: relative;
    overflow: hidden;
    min-width: 200px;
}

.progress-fill {
    height: 100%;
    background: linear-gradient(90deg, var(--accent-primary), var(--accent-hover));
    transition: width 0.3s ease;
    position: relative;
}

.progress-fill::after {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
    animation: shimmer 2s infinite;
}

@keyframes shimmer {
    0% { transform: translateX(-100%); }
    100% { transform: translateX(100%); }
}

.controls {
    display: flex;
    gap: 10px;
    align-items: center;
}

.search-input {
    background: var(--bg-primary);
    border: 1px solid var(--border);
    color: var(--text-primary);
    padding: 6px 12px;
    font-size: 0.85rem;
    font-family: inherit;
    outline: none;
    width: 200px;
}

.search-input:focus {
    border-color: var(--accent-primary);
}

.filter-btn, .export-btn {
    background: var(--bg-card);
    border: 1px solid var(--border);
    color: var(--text-primary);
    padding: 6px 12px;
    font-size: 0.75rem;
    font-family: inherit;
    cursor: pointer;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    transition: all 0.2s ease;
    border-radius: 0;
}

.filter-btn:hover, .export-btn:hover {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.filter-btn.active {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

/* Table */
.table-container {
    padding: 0;
    overflow-x: auto;
    height: calc(100vh - 50px);
}

table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.85rem;
}

thead {
    position: sticky;
    top: 0;
    z-index: 100;
    background: var(--bg-secondary);
}

th {
    padding: 10px 12px;
    text-align: left;
    font-weight: 700;
    text-transform: uppercase;
    font-size: 0.7rem;
    letter-spacing: 0.08em;
    border-bottom: 2px solid var(--accent-primary);
    border-right: 1px solid var(--border);
    background: var(--bg-secondary);
    white-space: nowrap;
}

th:last-child {
    border-right: none;
}

th.sortable {
    cursor: pointer;
    user-select: none;
}

th.sortable:hover {
    background: var(--bg-card);
}

th.sorted-asc::after {
    content: ' ▲';
    color: var(--accent-primary);
}

th.sorted-desc::after {
    content: ' ▼';
    color: var(--accent-primary);
}

tbody tr {
    border-bottom: 1px solid var(--border);
    transition: background 0.15s ease;
}

tbody tr:hover {
    background: var(--bg-card);
}

tbody tr.completed {
    opacity: 0.5;
    background: var(--bg-secondary);
}

tbody tr.completed:hover {
    opacity: 0.7;
}

td {
    padding: 8px 12px;
    border-right: 1px solid var(--border);
    vertical-align: middle;
}

td:last-child {
    border-right: none;
}

/* Checkbox */
.checkbox-cell {
    width: 40px;
    text-align: center;
    padding: 8px;
}

.task-checkbox {
    width: 18px;
    height: 18px;
    border: 2px solid var(--border);
    background: var(--bg-primary);
    cursor: pointer;
    position: relative;
    display: inline-block;
}

.task-checkbox.checked {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.task-checkbox.checked::after {
    content: '✓';
    position: absolute;
    top: -2px;
    left: 2px;
    font-size: 14px;
    color: var(--bg-primary);
    font-weight: 900;
}

/* Trader */
.trader-cell {
    width: 100px;
    font-weight: 600;
    color: var(--accent-primary);
}

/* Task Name */
.task-name-cell {
    min-width: 250px;
    font-weight: 600;
    cursor: pointer;
}

.task-name-cell:hover {
    color: var(--accent-hover);
}

/* Level */
.level-cell {
    width: 60px;
    text-align: center;
    color: var(--text-secondary);
    font-weight: 700;
}

/* Prerequisites */
.prereq-cell {
    width: 180px;
    font-size: 0.75rem;
}

.prereq-item {
    display: block;
    padding: 2px 0;
    color: var(--text-secondary);
}

.prereq-item.completed {
    color: var(--success);
    text-decoration: line-through;
}

.prereq-item.incomplete {
    color: var(--text-primary);
    font-weight: 600;
}

/* Unlocks */
.unlocks-cell {
    width: 80px;
    font-size: 0.75rem;
    text-align: center;
}

/* Hover count + list pattern */
.hover-count-wrapper {
    position: relative;
    display: inline-block;
}

.hover-count {
    display: inline-block;
    background: var(--bg-secondary);
    border: 1px solid var(--border);
    padding: 2px 8px;
    font-size: 0.75rem;
    font-weight: 700;
    color: var(--text-secondary);
    cursor: default;
    min-width: 28px;
    text-align: center;
}

.hover-list {
    display: none;
    position: absolute;
    top: 100%;
    left: 50%;
    transform: translateX(-50%);
    z-index: 500;
    background: var(--bg-card);
    border: 1px solid var(--border-light);
    padding: 8px 12px;
    min-width: 250px;
    max-width: 400px;
    max-height: 300px;
    overflow-y: auto;
    box-shadow: 0 4px 12px var(--shadow);
}

.hover-count-wrapper:hover .hover-list {
    display: block;
}

.hover-list-item {
    padding: 4px 0;
    border-bottom: 1px solid var(--border);
    font-size: 0.75rem;
    color: var(--text-secondary);
    line-height: 1.4;
}

.hover-list-item:last-child {
    border-bottom: none;
}

.hover-list-item.is-completed {
    color: var(--success);
    text-decoration: line-through;
    opacity: 0.6;
}

.hover-list-item.is-optional {
    color: var(--text-muted);
    font-style: italic;
}

.hover-list-item.is-optional::before {
    content: '(Optional) ';
    font-weight: 600;
}

.hover-list-item.is-fir {
    color: var(--success);
}

/* Map */
.map-cell {
    width: 120px;
    color: var(--text-secondary);
}

/* XP */
.xp-cell {
    width: 80px;
    text-align: right;
    color: var(--text-secondary);
    font-weight: 700;
}

/* Objectives */
.objectives-cell {
    width: 80px;
    font-size: 0.75rem;
    color: var(--text-secondary);
    text-align: center;
}

/* Items */
.items-cell {
    width: 80px;
    font-size: 0.75rem;
    text-align: center;
}

/* Badges */
.badges-cell {
    width: 100px;
    text-align: center;
}

.badge {
    display: inline-block;
    padding: 2px 6px;
    font-size: 0.65rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    margin: 0 2px;
}

.badge.kappa {
    background: var(--kappa);
    color: var(--bg-primary);
}

.badge.lightkeeper {
    background: var(--lightkeeper);
    color: var(--text-primary);
}

/* Wiki */
.wiki-cell {
    width: 60px;
    text-align: center;
}

.wiki-link {
    color: var(--accent-primary);
    text-decoration: none;
    font-size: 0.7rem;
    text-transform: uppercase;
    font-weight: 700;
    padding: 4px 8px;
    border: 1px solid var(--accent-primary);
    display: inline-block;
    transition: all 0.2s ease;
}

.wiki-link:hover {
    background: var(--accent-primary);
    color: var(--bg-primary);
}

/* Loading & Error */
#loading, #error {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    padding: 20px;
    background: var(--bg-secondary);
    border: 2px solid var(--accent-primary);
    font-size: 1rem;
    z-index: 2000;
}

#error {
    border-color: #d9534f;
    color: #d9534f;
}

/* Grid View */
.grid-view {
    display: none;
    padding: 20px;
    background: var(--bg-primary);
}

.grid-view.active {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 20px;
}

.trader-grid-section {
    background: var(--bg-secondary);
    border: 1px solid var(--border);
    padding: 15px;
}

.trader-grid-header {
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--accent-primary);
    font-size: 0.9rem;
    margin-bottom: 12px;
    padding-bottom: 8px;
    border-bottom: 2px solid var(--accent-primary);
}

.trader-grid-stats {
    font-size: 0.7rem;
    color: var(--text-muted);
    margin-top: 4px;
}

.grid-task-item {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 6px 0;
    font-size: 0.85rem;
    border-bottom: 1px solid var(--border);
}

.grid-task-item:last-child {
    border-bottom: none;
}

.grid-task-item.completed {
    opacity: 0.4;
}

.grid-task-checkbox {
    width: 16px;
    height: 16px;
    border: 2px solid var(--border);
    background: var(--bg-primary);
    cursor: pointer;
    flex-shrink: 0;
    position: relative;
}

.grid-task-checkbox.checked {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.grid-task-checkbox.checked::after {
    content: '✓';
    position: absolute;
    top: -2px;
    left: 1px;
    font-size: 12px;
    color: var(--bg-primary);
    font-weight: 900;
}

.grid-task-name {
    flex: 1;
    cursor: pointer;
    word-break: break-word;
}

.grid-task-name:hover {
    color: var(--accent-hover);
}

.grid-task-badges {
    display: flex;
    gap: 4px;
}

.grid-badge {
    font-size: 0.6rem;
    padding: 2px 4px;
    font-weight: 700;
}

/* ====== Maps Page ====== */
.maps-page {
    height: 100vh;
    display: flex;
    flex-direction: column;
}

.maps-header {
    background: var(--bg-secondary);
    border-bottom: 2px solid var(--border);
    padding: 12px 20px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 20px;
    box-shadow: 0 2px 10px var(--shadow);
}

.maps-tab-bar {
    display: flex;
    gap: 4px;
    flex-wrap: wrap;
}

.map-tab {
    background: var(--bg-card);
    border: 1px solid var(--border);
    color: var(--text-primary);
    padding: 6px 14px;
    font-size: 0.75rem;
    font-family: inherit;
    font-weight: 700;
    cursor: pointer;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    transition: all 0.2s ease;
    border-radius: 0;
}

.map-tab:hover {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.map-tab.active {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.map-view-toggle {
    display: flex;
    gap: 4px;
}

.map-view-btn {
    background: var(--bg-card);
    border: 1px solid var(--border);
    color: var(--text-primary);
    padding: 6px 14px;
    font-size: 0.75rem;
    font-family: inherit;
    font-weight: 700;
    cursor: pointer;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    transition: all 0.2s ease;
    border-radius: 0;
}

.map-view-btn:hover {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.map-view-btn.active {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.maps-header-right {
    display: flex;
    align-items: center;
    gap: 10px;
}

.map-info-btn {
    background: var(--bg-card);
    border: 1px solid var(--border);
    color: var(--text-primary);
    padding: 6px 14px;
    font-size: 0.75rem;
    font-family: inherit;
    font-weight: 700;
    cursor: pointer;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    transition: all 0.2s ease;
    border-radius: 0;
}

.map-info-btn:hover,
.map-info-btn.active {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.maps-content {
    flex: 1;
    display: flex;
    align-items: stretch;
    justify-content: center;
    background: var(--bg-primary);
    position: relative;
    overflow: hidden;
}

.maps-placeholder {
    font-size: 1.2rem;
    color: var(--text-muted);
    text-transform: uppercase;
    letter-spacing: 0.15em;
    font-weight: 700;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
}

/* Map iframe */
.map-iframe {
    width: 100%;
    height: 100%;
    border: 0;
}

/* Map image */
.map-image-container {
    width: 100%;
    height: 100%;
    overflow: auto;
    display: flex;
    align-items: flex-start;
    justify-content: center;
    background: var(--bg-primary);
}

.map-image {
    max-width: 100%;
    height: auto;
    display: block;
}

/* Map Info Panel */
.map-info-panel {
    position: absolute;
    top: 0;
    right: 0;
    width: 320px;
    max-height: 100%;
    overflow-y: auto;
    background: var(--bg-secondary);
    border-left: 2px solid var(--border);
    padding: 16px;
    z-index: 200;
    box-shadow: -4px 0 12px var(--shadow);
}

.map-info-title {
    font-size: 1rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--accent-primary);
    margin-bottom: 8px;
    padding-bottom: 8px;
    border-bottom: 2px solid var(--accent-primary);
}

.map-info-desc {
    font-size: 0.75rem;
    color: var(--text-secondary);
    line-height: 1.5;
    margin-bottom: 12px;
}

.map-info-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px;
    margin-bottom: 12px;
}

.map-info-item {
    background: var(--bg-card);
    border: 1px solid var(--border);
    padding: 8px;
}

.map-info-label {
    display: block;
    font-size: 0.6rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--text-muted);
    margin-bottom: 2px;
}

.map-info-value {
    font-size: 0.85rem;
    font-weight: 700;
    color: var(--text-primary);
}

.map-info-section {
    margin-bottom: 12px;
}

.map-info-section-title {
    font-size: 0.65rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--text-muted);
    margin-bottom: 6px;
}

.map-info-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 4px;
}

.map-info-tag {
    background: var(--bg-card);
    border: 1px solid var(--border);
    padding: 2px 8px;
    font-size: 0.7rem;
    color: var(--text-secondary);
}

.map-info-boss {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 4px 0;
    border-bottom: 1px solid var(--border);
    font-size: 0.8rem;
}

.map-info-boss:last-child {
    border-bottom: none;
}

.boss-name {
    color: var(--text-primary);
    font-weight: 600;
}

.boss-chance {
    color: var(--kappa);
    font-weight: 700;
    font-size: 0.75rem;
}

/* ====== Tarkov Time Widget ====== */
.tarkov-time-widget {
    position: fixed;
    bottom: 24px;
    left: 24px;
    z-index: 2000;
    background: var(--bg-secondary);
    border: 1px solid var(--border);
    padding: 10px 14px;
    min-width: 150px;
    box-shadow: 0 4px 12px var(--shadow);
    user-select: none;
}

.tarkov-time-header {
    font-size: 0.6rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.12em;
    color: var(--text-muted);
    margin-bottom: 6px;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.time-reset {
    color: var(--accent-primary);
    cursor: pointer;
    font-size: 0.55rem;
}

.time-reset:hover {
    color: var(--accent-hover);
}

.tarkov-time-row {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 2px 0;
}

.time-label {
    font-size: 0.7rem;
    font-weight: 700;
    color: var(--text-muted);
    width: 16px;
}

.time-value {
    font-size: 0.9rem;
    font-weight: 700;
    color: var(--accent-primary);
    font-variant-numeric: tabular-nums;
}

.tarkov-time-offsets {
    margin-top: 8px;
    padding-top: 6px;
    border-top: 1px solid var(--border);
}

.time-until-item {
    display: flex;
    justify-content: space-between;
    padding: 1px 0;
}

.time-until-label {
    font-size: 0.65rem;
    color: var(--text-muted);
}

.time-until-value {
    font-size: 0.65rem;
    color: var(--text-secondary);
    font-weight: 700;
    font-variant-numeric: tabular-nums;
}

.time-offset-hint {
    font-size: 0.55rem;
    color: var(--text-muted);
    margin-top: 4px;
    text-align: center;
    font-style: italic;
}

/* ====== FAB Navigation ====== */
.fab-container {
    position: fixed;
    bottom: 24px;
    right: 24px;
    z-index: 2000;
    display: flex;
    flex-direction: column-reverse;
    align-items: center;
    gap: 12px;
}

.fab-button {
    width: 50px;
    height: 50px;
    border-radius: 50%;
    background: var(--accent-primary);
    border: 2px solid var(--accent-hover);
    color: var(--bg-primary);
    font-size: 1.5rem;
    font-weight: 900;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 4px 12px var(--shadow);
    transition: background 0.2s ease, transform 0.2s ease;
    font-family: inherit;
    padding: 0;
}

.fab-button:hover {
    background: var(--accent-hover);
    transform: scale(1.05);
}

.fab-icon {
    display: inline-block;
    transition: transform 0.3s ease;
    line-height: 1;
}

.fab-icon.open {
    transform: rotate(45deg);
}

.fab-children {
    display: flex;
    flex-direction: column-reverse;
    align-items: center;
    gap: 10px;
}

.fab-child-btn {
    width: 42px;
    height: 42px;
    border-radius: 50%;
    background: var(--bg-card);
    border: 2px solid var(--border);
    color: var(--text-primary);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 2px 8px var(--shadow);
    transition: all 0.2s ease;
    padding: 0;
}

.fab-child-btn:hover {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
    color: var(--bg-primary);
}

.fab-child-btn.active {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
    color: var(--bg-primary);
}

.fab-child-enter-active,
.fab-child-leave-active {
    transition: all 0.25s ease;
}

.fab-child-enter-from,
.fab-child-leave-to {
    opacity: 0;
    transform: translateY(20px) scale(0.5);
}

@media (max-width: 1200px) {
    .objectives-cell,
    .items-cell {
        min-width: auto;
    }
}

/* Below Level Styling */
tbody tr.below-level {
    opacity: 0.4;
    background: var(--bg-secondary);
}

tbody tr.below-level:hover {
    opacity: 0.5;
}

.grid-task-item.below-level {
    opacity: 0.4;
}

/* ====== Modal Styles ====== */
.modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.8);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 3000;
    padding: 20px;
}

.modal-content {
    background: var(--bg-secondary);
    border: 2px solid var(--border);
    max-width: 600px;
    width: 100%;
    max-height: 90vh;
    overflow-y: auto;
    box-shadow: 0 8px 32px var(--shadow);
}

.modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px 20px;
    border-bottom: 2px solid var(--accent-primary);
    background: var(--bg-card);
}

.modal-header h2 {
    margin: 0;
    font-size: 1rem;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--accent-primary);
}

.modal-close {
    background: none;
    border: none;
    color: var(--text-secondary);
    font-size: 1.5rem;
    cursor: pointer;
    padding: 0;
    line-height: 1;
}

.modal-close:hover {
    color: var(--text-primary);
}

.modal-body {
    padding: 20px;
}

.modal-footer {
    padding: 16px 20px;
    border-top: 1px solid var(--border);
    display: flex;
    justify-content: flex-end;
    gap: 10px;
}

/* Drop Zone */
.drop-zone {
    border: 2px dashed var(--border);
    padding: 40px 20px;
    text-align: center;
    transition: all 0.2s ease;
    min-height: 200px;
    display: flex;
    align-items: center;
    justify-content: center;
}

.drop-zone.drag-over {
    border-color: var(--accent-primary);
    background: rgba(125, 138, 60, 0.1);
}

.drop-zone.has-files {
    padding: 15px;
    min-height: auto;
}

.drop-zone-content {
    color: var(--text-muted);
}

.drop-zone-content svg {
    margin-bottom: 15px;
    opacity: 0.5;
}

.drop-zone-content p {
    margin: 0 0 8px 0;
}

.drop-zone-hint {
    font-size: 0.75rem;
    color: var(--text-muted);
}

/* File List */
.file-list {
    width: 100%;
}

.file-item {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 10px;
    background: var(--bg-card);
    border: 1px solid var(--border);
    margin-bottom: 8px;
}

.file-item:last-child {
    margin-bottom: 0;
}

.file-preview {
    width: 50px;
    height: 50px;
    object-fit: cover;
    border: 1px solid var(--border);
}

.file-icon {
    width: 50px;
    height: 50px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-secondary);
    border: 1px solid var(--border);
    font-size: 0.7rem;
    font-weight: 700;
    color: var(--accent-primary);
}

.file-info {
    flex: 1;
}

.file-name {
    font-size: 0.85rem;
    color: var(--text-primary);
    word-break: break-all;
}

.file-status {
    font-size: 0.7rem;
    text-transform: uppercase;
    margin-top: 4px;
}

.file-status.pending {
    color: var(--text-muted);
}

.file-status.processing {
    color: var(--kappa);
}

.file-status.done {
    color: var(--success);
}

.file-status.error {
    color: #d9534f;
}

.file-remove {
    background: none;
    border: none;
    color: var(--text-muted);
    font-size: 1.2rem;
    cursor: pointer;
    padding: 5px;
}

.file-remove:hover {
    color: #d9534f;
}

/* Import Actions */
.import-actions {
    display: flex;
    gap: 10px;
    margin-top: 15px;
}

.import-btn {
    background: var(--bg-card);
    border: 1px solid var(--border);
    color: var(--text-primary);
    padding: 10px 20px;
    font-size: 0.8rem;
    font-family: inherit;
    cursor: pointer;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    transition: all 0.2s ease;
}

.import-btn:hover {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.import-btn.primary {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.import-btn.primary:hover {
    background: var(--accent-hover);
    border-color: var(--accent-hover);
}

.import-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

/* Import Progress */
.import-progress {
    height: 4px;
    background: var(--bg-primary);
    margin-top: 15px;
    overflow: hidden;
}

.import-progress-bar {
    height: 100%;
    background: var(--accent-primary);
    transition: width 0.3s ease;
}

/* Import Results */
.import-results {
    margin-top: 20px;
    border: 1px solid var(--border);
    background: var(--bg-card);
}

.results-header {
    font-size: 0.7rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--text-muted);
    padding: 10px 12px;
    border-bottom: 1px solid var(--border);
}

.result-item {
    padding: 8px 12px;
    border-bottom: 1px solid var(--border);
    font-size: 0.8rem;
    display: flex;
    gap: 10px;
}

.result-item:last-child {
    border-bottom: none;
}

.result-type {
    font-weight: 700;
    color: var(--accent-primary);
    font-size: 0.7rem;
    min-width: 50px;
}

.result-message {
    color: var(--text-secondary);
}

/* Import Example */
.import-example {
    margin-top: 20px;
    border: 1px solid var(--border);
    background: var(--bg-card);
}

.example-header {
    font-size: 0.7rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--text-muted);
    padding: 10px 12px;
    border-bottom: 1px solid var(--border);
}

.example-content {
    padding: 12px;
    font-size: 0.8rem;
    color: var(--text-secondary);
}

.example-content p {
    margin: 0 0 10px 0;
}

.example-content ul {
    margin: 0;
    padding-left: 20px;
}

.example-content li {
    margin-bottom: 6px;
}

.status-completed {
    color: var(--success);
    font-weight: 600;
}

.status-active {
    color: var(--kappa);
    font-weight: 600;
}

.status-locked {
    color: var(--text-muted);
    font-weight: 600;
}

/* ====== Settings Modal ====== */
.settings-modal {
    max-width: 500px;
}

.settings-section {
    margin-bottom: 24px;
}

.settings-section:last-child {
    margin-bottom: 0;
}

.settings-section-title {
    font-size: 0.8rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--accent-primary);
    margin-bottom: 12px;
    padding-bottom: 8px;
    border-bottom: 1px solid var(--border);
}

.settings-field {
    margin-bottom: 15px;
}

.settings-field label {
    display: block;
    font-size: 0.75rem;
    color: var(--text-secondary);
    margin-bottom: 6px;
    text-transform: uppercase;
    letter-spacing: 0.05em;
}

.input-row {
    display: flex;
    gap: 10px;
}

.settings-input {
    flex: 1;
    background: var(--bg-primary);
    border: 1px solid var(--border);
    color: var(--text-primary);
    padding: 10px 12px;
    font-size: 0.85rem;
    font-family: inherit;
    outline: none;
}

.settings-input:focus {
    border-color: var(--accent-primary);
}

.settings-btn {
    background: var(--bg-card);
    border: 1px solid var(--border);
    color: var(--text-primary);
    padding: 10px 16px;
    font-size: 0.8rem;
    font-family: inherit;
    cursor: pointer;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    transition: all 0.2s ease;
}

.settings-btn:hover {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.settings-btn.primary {
    background: var(--accent-primary);
    border-color: var(--accent-primary);
}

.settings-btn.primary:hover {
    background: var(--accent-hover);
    border-color: var(--accent-hover);
}

.settings-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.settings-hint {
    font-size: 0.7rem;
    color: var(--text-muted);
    margin-top: 6px;
}

.settings-hint a {
    color: var(--accent-primary);
    text-decoration: none;
}

.settings-hint a:hover {
    text-decoration: underline;
}

.settings-error {
    color: #d9534f;
    font-size: 0.8rem;
    margin-top: 10px;
    padding: 8px;
    background: rgba(217, 83, 79, 0.1);
    border: 1px solid rgba(217, 83, 79, 0.3);
}

/* Player Info Card */
.player-info-card {
    background: var(--bg-card);
    border: 1px solid var(--border);
    padding: 15px;
    margin-top: 15px;
}

.player-info-row {
    display: flex;
    justify-content: space-between;
    padding: 6px 0;
    border-bottom: 1px solid var(--border);
    font-size: 0.85rem;
}

.player-info-row:last-child {
    border-bottom: none;
}

.player-label {
    color: var(--text-muted);
}

.player-value {
    color: var(--text-primary);
    font-weight: 600;
}

.player-level {
    color: var(--accent-primary);
    font-size: 1.1rem;
}

/* Radio Group */
.radio-group {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-top: 12px;
}

.radio-option {
    display: flex;
    align-items: flex-start;
    gap: 10px;
    padding: 12px;
    background: var(--bg-card);
    border: 1px solid var(--border);
    cursor: pointer;
    transition: all 0.2s ease;
}

.radio-option:hover {
    border-color: var(--accent-primary);
}

.radio-option.active {
    border-color: var(--accent-primary);
    background: rgba(125, 138, 60, 0.1);
}

.radio-option input {
    margin: 0;
    margin-top: 2px;
}

.radio-label {
    display: block;
    font-size: 0.85rem;
    font-weight: 600;
    color: var(--text-primary);
}

.radio-desc {
    display: block;
    font-size: 0.7rem;
    color: var(--text-muted);
    margin-top: 2px;
}
</style>
