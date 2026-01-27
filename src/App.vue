<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const API_URL = 'https://api.tarkov.dev/graphql'
const STORAGE_KEY = 'tarkov_kappa_tracker_v2'
const CACHE_KEY = 'tarkov_kappa_task_cache'
const MAP_CACHE_KEY = 'tarkov_map_cache'

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
  'customs': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/5/55/Customs-3d-by-loweffort-v2.png/revision/latest',
  'factory': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/e/ea/Factory_2D_by_Jindouz.png/revision/latest',
  'ground-zero': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/d/d2/Ground_Zero_2D.png/revision/latest',
  'interchange': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/3/3b/Interchange_2D_by_Jindouz.png/revision/latest',
  'lighthouse': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/1/13/Jindouz_Lighthouse_Map_V1.png/revision/latest',
  'reserve': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/f/f1/Reserve_Map_Nitro_V3.png/revision/latest',
  'shoreline': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/e/e5/Shoreline_2D_by_Jindouz.png/revision/latest',
  'streets-of-tarkov': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/c/c4/Streets_2D_by_Jindouz.png/revision/latest',
  'the-lab': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/b/bb/TheLabMapByMonkiAK.png/revision/latest',
  'woods': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/4/4c/Woods_2D_by_Jindouz.png/revision/latest',
}

// 3D map images by normalizedName
const map3dUrls = {
  'customs': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/5/55/Customs-3d-by-loweffort-v2.png/revision/latest',
  'factory': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/8/80/Factory-3d-by-loweffort.png/revision/latest',
  'ground-zero': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/d/d2/Ground_Zero_2D.png/revision/latest',
  'interchange': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/1/18/Interchange-3d-by-loweffort.png/revision/latest',
  'lighthouse': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/9/9f/Re3mrLighthouseVerticalDay-min.jpg/revision/latest',
  'reserve': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/5/54/Reserve-3d-by-loweffort.png/revision/latest',
  'shoreline': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/2/25/Shoreline-3d-by-loweffort.png/revision/latest',
  'streets-of-tarkov': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/4/40/Streets-3d-by-loweffort.png/revision/latest',
  'the-lab': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/b/bb/TheLabMapByMonkiAK.png/revision/latest',
  'woods': 'https://static.wikia.nocookie.net/escapefromtarkov_gamepedia/images/c/cc/Woods-3d-by-loweffort.png/revision/latest',
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
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = 'application/json'
  input.onchange = (e) => {
    const file = e.target.files[0]
    const reader = new FileReader()
    reader.onload = (event) => {
      try {
        const data = JSON.parse(event.target.result)
        completedTasks.value = new Set(data.completed || [])
        saveProgress()
        alert('Data imported successfully!')
      } catch (err) {
        alert('Error importing data: ' + err.message)
      }
    }
    reader.readAsText(file)
  }
  input.click()
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
            :class="{ completed: isCompleted(task.id) }"
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
          :class="{ completed: isCompleted(task.id) }"
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
      </div>
    </transition>
    <button class="fab-button" @click="fabOpen = !fabOpen">
      <span class="fab-icon" :class="{ open: fabOpen }">+</span>
    </button>
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
</style>
