<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue'
import { supabase } from './supabase'

const boardWidth = ref(30)
const boardHeight = ref(30)
type EntityType = 'rally' | 'joiner' | 'member' | 'bear_trap' | 'flag' | 'hq' | 'farm' | 'statue' | 'mountain'

interface Entity {
  id: string
  type: EntityType
  x: number
  y: number
  name?: string
  cityNumber?: number
}

const entityDefs: Record<EntityType, { label: string, class: string, colorClass: string, icon?: string, w: number, h: number }> = {
  rally: { label: 'Rally', class: 'bg-rose-300 text-slate-800 rounded-sm', colorClass: 'bg-rose-300', w: 2, h: 2 },
  joiner: { label: 'Joiner', class: 'bg-orange-300 text-slate-800 rounded-sm', colorClass: 'bg-orange-300', w: 2, h: 2 },
  member: { label: 'Member', class: 'bg-slate-300 text-slate-800 rounded-sm', colorClass: 'bg-slate-300', w: 2, h: 2 },
  bear_trap: { label: 'Bear Trap', class: 'bg-indigo-300 text-slate-800 rounded-sm flex items-center justify-center text-xl', colorClass: 'bg-indigo-300', icon: '🐻', w: 3, h: 3 },
  flag: { label: 'Flag', class: 'bg-blue-300 text-slate-800 rounded-sm flex items-center justify-center text-[10px]', colorClass: 'bg-blue-300', icon: '🚩', w: 1, h: 1 },
  hq: { label: 'HQ', class: 'bg-violet-300 text-slate-800 rounded-sm flex items-center justify-center text-xl', colorClass: 'bg-violet-300', icon: '🏰', w: 3, h: 3 },
  farm: { label: 'Farm', class: 'bg-amber-300 text-slate-800 rounded-sm flex items-center justify-center text-lg', colorClass: 'bg-amber-300', icon: '🌾', w: 2, h: 2 },
  statue: { label: 'Statue', class: 'bg-stone-300 text-slate-800 rounded-sm flex items-center justify-center text-lg', colorClass: 'bg-stone-300', icon: '🗿', w: 2, h: 2 },
  mountain: { label: 'Mountain', class: 'bg-slate-400 text-slate-800 rounded-sm flex items-center justify-center text-[10px]', colorClass: 'bg-slate-400', icon: '⛰️', w: 1, h: 1 },
}

const entities = ref<Entity[]>([])
const hideEffectArea = ref(false)
const selectedEntityId = ref<string | null>(null)
const hoveredEntityId = ref<string | null>(null) // Canvas hover
const listHoveredEntityId = ref<string | null>(null) // List hover
const cityCounter = ref(1)
const editingEntityId = ref<string | null>(null)

// Drag and Drop States
const dragX = ref<number | null>(null)
const dragY = ref<number | null>(null)
const dragType = ref<EntityType | null>(null)
const dragMoveId = ref<string | null>(null)

const gridCellsMap = computed(() => {
  const map: Record<string, string> = {}
  entities.value.forEach(e => {
    const def = entityDefs[e.type]
    for (let i = 0; i < def.w; i++) {
        for (let j = 0; j < def.h; j++) {
            map[`${e.x + i},${e.y + j}`] = e.id
        }
    }
  })
  return map
})

const citiesList = computed(() => {
  return entities.value.filter(e => ['rally', 'joiner', 'member'].includes(e.type)).sort((a, b) => (a.cityNumber || 0) - (b.cityNumber || 0))
})

const isSpaceFree = (x: number, y: number, w: number, h: number, ignoreId?: string) => {
  if (x < 0 || y < 0 || x + w > boardWidth.value || y + h > boardHeight.value) return false
  for (let i = 0; i < w; i++) {
    for (let j = 0; j < h; j++) {
      const occupiedBy = gridCellsMap.value[`${x + i},${y + j}`]
      if (occupiedBy && occupiedBy !== ignoreId) return false
    }
  }
  return true
}

const addEntity = (type: EntityType, x: number, y: number, checkOverlap = true, showErrors = true) => {
  const def = entityDefs[type]
  let targetX = x
  let targetY = y

  if (targetX + def.w > boardWidth.value) targetX = boardWidth.value - def.w
  if (targetY + def.h > boardHeight.value) targetY = boardHeight.value - def.h
  if (targetX < 0) targetX = 0
  if (targetY < 0) targetY = 0

  if (type === 'bear_trap') {
    const traps = entities.value.filter(e => e.type === 'bear_trap').length
    if (traps >= 2) {
      if (showErrors) alert('Max of 2 Bear Traps allowed!')
      return false
    }
  }

  if (checkOverlap && !isSpaceFree(targetX, targetY, def.w, def.h)) {
    if (showErrors) alert('Space is occupied!')
    return false
  }

  const isCity = ['rally', 'joiner', 'member'].includes(type)
  const id = Math.random().toString(36).substring(2, 9)
  let name = undefined
  let cityNumber = undefined
  if (isCity) {
    cityNumber = cityCounter.value++
    name = `City ${cityNumber}`
  }
  entities.value.push({ id, type, x: targetX, y: targetY, name, cityNumber })
  return true
}

const recalculateCityNumbers = () => {
    let count = 1
    const cities = entities.value
        .filter(e => ['rally', 'joiner', 'member'].includes(e.type))
        .sort((a, b) => (a.cityNumber || 0) - (b.cityNumber || 0))
    
    cities.forEach(city => {
        city.cityNumber = count++
        // Only update name if it follows the default pattern
        if (city.name && /^City \d+$/.test(city.name)) {
            city.name = `City ${city.cityNumber}`
        }
    })
    cityCounter.value = count
}

const deleteEntity = (id: string) => {
    const entity = entities.value.find(e => e.id === id)
    if (!entity) return
    const wasCity = ['rally', 'joiner', 'member'].includes(entity.type)
    entities.value = entities.value.filter(ent => ent.id !== id)
    if (wasCity) {
        recalculateCityNumbers()
    }
}

watch([boardWidth, boardHeight], () => {
  entities.value.forEach(ent => {
    const def = entityDefs[ent.type]
    if (ent.x + def.w > boardWidth.value) ent.x = Math.max(0, boardWidth.value - def.w)
    if (ent.y + def.h > boardHeight.value) ent.y = Math.max(0, boardHeight.value - def.h)
  })
})

const moveEntity = (id: string, newX: number, newY: number) => {
  const entity = entities.value.find(e => e.id === id)
  if (!entity) return
  const def = entityDefs[entity.type]
  
  let targetX = newX
  let targetY = newY

  if (targetX + def.w > boardWidth.value) targetX = boardWidth.value - def.w
  if (targetY + def.h > boardHeight.value) targetY = boardHeight.value - def.h
  if (targetX < 0) targetX = 0
  if (targetY < 0) targetY = 0

  if (isSpaceFree(targetX, targetY, def.w, def.h, id)) {
    entity.x = targetX
    entity.y = targetY
  }
}

const placeEntity = (type: EntityType) => {
  const def = entityDefs[type]
  for (let y = 0; y < boardHeight.value; y++) {
    for (let x = 0; x < boardWidth.value; x++) {
      if (isSpaceFree(x, y, def.w, def.h)) {
        addEntity(type, x, y, false)
        return
      }
    }
  }
  alert('No free space available for this item!')
}

const handleDragStartSidebar = (e: DragEvent, type: EntityType) => {
  if (e.dataTransfer) {
    e.dataTransfer.setData('action', 'create')
    e.dataTransfer.setData('type', type)
    e.dataTransfer.effectAllowed = 'copy'
    dragType.value = type
    dragMoveId.value = null
  }
}

const handleDragStartBoard = (e: DragEvent, id: string) => {
  const entity = entities.value.find(ent => ent.id === id)
  if (e.dataTransfer && entity) {
    e.dataTransfer.setData('action', 'move')
    e.dataTransfer.setData('id', id)
    e.dataTransfer.effectAllowed = 'move'
    dragMoveId.value = id
    dragType.value = entity.type
    
    const canvas = document.createElement('canvas')
    canvas.width = 1
    canvas.height = 1
    e.dataTransfer.setDragImage(canvas, 0, 0)
  }
}

const handleDragOver = (e: DragEvent, x: number, y: number) => {
  e.preventDefault()
  if (e.dataTransfer) {
     e.dataTransfer.dropEffect = 'move'
     dragX.value = x
     dragY.value = y
  }
}

const handleDragEnd = () => {
    dragX.value = null
    dragY.value = null
    dragType.value = null
    dragMoveId.value = null
}

const handleDrop = (e: DragEvent, x: number, y: number) => {
  e.preventDefault()
  const action = e.dataTransfer?.getData('action')
  if (action === 'create') {
    const type = e.dataTransfer?.getData('type') as EntityType
    if (type) addEntity(type, x, y, true, true)
  } else if (action === 'move') {
    const id = e.dataTransfer?.getData('id')
    if (id) moveEntity(id, x, y)
  }
  handleDragEnd()
}

const selectEntity = (id: string | null) => {
  selectedEntityId.value = id
}

const getSelectedEntity = computed(() => {
  return entities.value.find(e => e.id === selectedEntityId.value)
})

const getHoveredEntity = computed(() => {
  const targetId = listHoveredEntityId.value || hoveredEntityId.value || selectedEntityId.value
  return entities.value.find(e => e.id === targetId)
})

const handleKeyDown = (e: KeyboardEvent) => {
  if (editingEntityId.value) return // Don't delete while editing name
  if (e.key === 'Delete' || e.key === 'Backspace') {
    if (selectedEntityId.value) {
      deleteEntity(selectedEntityId.value)
      selectedEntityId.value = null
    }
  }
}

const clickGridCell = (x: number, y: number) => {
  const clickedId = gridCellsMap.value[`${x},${y}`] || null
  selectEntity(clickedId)
}

const currentConfigId = ref<string | null>(null)
const currentConfigSlug = ref<string | null>(null)
const isAdmin = ref(true)
const showSaveModal = ref(false)
const showShareModal = ref(false)
const savePassword = ref('')
const generatedSlug = ref('')
const isSaving = ref(false)

const loadConfigBySlug = async (slug: string) => {
  const { data: config, error } = await supabase
    .from('board_configurations')
    .select('*')
    .eq('read_only_slug', slug)
    .single()

  if (error || !config) {
    alert('Configuration not found!')
    return
  }

  if (Array.isArray(config.data)) {
    entities.value = config.data
    boardWidth.value = 30
    boardHeight.value = 30
  } else if (config.data && typeof config.data === 'object') {
    entities.value = config.data.entities || []
    boardWidth.value = config.data.width || 30
    boardHeight.value = config.data.height || 30
  }
  
  currentConfigId.value = config.id
  currentConfigSlug.value = config.read_only_slug
  isAdmin.value = false // Default to read-only when loading via slug
}

onMounted(async () => {
  window.addEventListener('keydown', handleKeyDown)
  
  const urlParams = new URLSearchParams(window.location.search)
  const slug = urlParams.get('b')
  if (slug) {
    await loadConfigBySlug(slug)
  }
})

const unlockAdmin = async () => {
  const password = prompt('Enter Admin Password:')
  if (!password || !currentConfigId.value) return

  const { data: config, error } = await supabase
    .from('board_configurations')
    .select('admin_password')
    .eq('id', currentConfigId.value)
    .single()

  if (error || !config) return

  if (config.admin_password === password) {
    isAdmin.value = true
    alert('Admin access granted!')
  } else {
    alert('Wrong password!')
  }
}

const saveConfig = async () => {
  if (!savePassword.value && !currentConfigId.value) {
    alert('Please enter a password to protect this board.')
    return
  }

  isSaving.value = true
  const payload = {
    entities: entities.value,
    width: boardWidth.value,
    height: boardHeight.value
  }

  try {
    if (currentConfigId.value && isAdmin.value) {
      // Update existing
      const { error } = await supabase
        .from('board_configurations')
        .update({ data: payload, updated_at: new Date() })
        .eq('id', currentConfigId.value)
      
      if (error) throw error
      alert('Updated successfully!')
    } else {
      // Create new
      const { data, error } = await supabase
        .from('board_configurations')
        .insert([{ 
          data: payload, 
          admin_password: savePassword.value 
        }])
        .select()
        .single()

      if (error) throw error
      
      currentConfigId.value = data.id
      currentConfigSlug.value = data.read_only_slug
      generatedSlug.value = data.read_only_slug
      showSaveModal.value = false
      showShareModal.value = true
    }
  } catch (e: any) {
    alert('Error saving: ' + e.message)
  } finally {
    isSaving.value = false
  }
}

const shareLink = computed(() => {
  const base = window.location.origin + window.location.pathname
  return `${base}?b=${currentConfigSlug.value || generatedSlug.value}`
})

const copyToClipboard = (text: string) => {
  navigator.clipboard.writeText(text)
  alert('Copied to clipboard!')
}

const showSidebar = ref(false)
const showCitiesList = ref(false)

const viewportScale = ref(1)
const zoomIn = () => { if(viewportScale.value < 2) viewportScale.value += 0.1 }
const zoomOut = () => { if(viewportScale.value > 0.5) viewportScale.value -= 0.1 }

</script>

<template>
  <div class="flex flex-col lg:flex-row h-screen bg-slate-100 text-slate-800 font-sans overflow-hidden">
    
    <!-- Header for Mobile -->
    <header class="lg:hidden bg-white border-b border-slate-200 p-4 flex items-center justify-between z-30">
      <h1 class="font-bold text-lg">Bear Space</h1>
      <div class="flex gap-2">
         <button @click="showSidebar = !showSidebar; showCitiesList = false" class="p-2 bg-slate-100 rounded-sm text-xs border">
           {{ showSidebar ? '✕ Close' : 'Tools' }}
         </button>
         <button @click="showCitiesList = !showCitiesList; showSidebar = false" class="p-2 bg-slate-100 rounded-sm text-xs border">
           {{ showCitiesList ? '✕ Close' : 'Cities' }}
         </button>
      </div>
    </header>

    <!-- Sidebar Tools -->
    <aside :class="['w-full lg:w-64 bg-white border-r border-slate-200 p-4 flex flex-col gap-6 overflow-y-auto lg:shrink-0 transition-all duration-300 z-20 shadow-none', 
                    showSidebar ? 'fixed inset-x-0 top-[65px] bottom-0 lg:static' : 'hidden lg:flex']">
      <div>
        <h3 class="text-xs font-semibold text-slate-400 mb-2 uppercase tracking-wider">Board Settings</h3>
        <div class="space-y-4 p-3 bg-slate-50 rounded-sm border border-slate-100">
           <div class="flex flex-col gap-1">
              <label class="text-[10px] text-slate-500 font-bold uppercase">Grid Width ({{boardWidth}})</label>
              <input type="range" v-model.number="boardWidth" min="10" max="60" step="5" class="w-full h-1 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-slate-800" />
           </div>
           <div class="flex flex-col gap-1">
              <label class="text-[10px] text-slate-500 font-bold uppercase">Grid Height ({{boardHeight}})</label>
              <input type="range" v-model.number="boardHeight" min="10" max="60" step="5" class="w-full h-1 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-slate-800" />
           </div>
        </div>
      </div>

      <div>
        <h3 class="text-xs font-semibold text-slate-400 mb-2 uppercase tracking-wider">Cities</h3>
        <div class="grid grid-cols-2 lg:grid-cols-1 gap-2">
          <div v-for="type in (['rally', 'joiner', 'member'] as const)" :key="type"
               @click="placeEntity(type); showSidebar = false" draggable="true" @dragstart="handleDragStartSidebar($event, type)" @dragend="handleDragEnd"
               class="flex items-center gap-2 lg:gap-3 p-2 bg-slate-100 rounded-sm cursor-pointer border border-transparent hover:border-slate-300">
            <div :class="entityDefs[type].colorClass" class="w-4 h-4 rounded-sm"></div>
            <span class="text-xs lg:text-sm font-medium">{{ entityDefs[type].label }}</span>
          </div>
        </div>
      </div>
      <div>
        <h3 class="text-xs font-semibold text-slate-400 mb-2 uppercase tracking-wider">Structures</h3>
        <div class="grid grid-cols-2 lg:grid-cols-1 gap-2">
          <div v-for="type in (['bear_trap', 'flag', 'hq'] as const)" :key="type"
               @click="placeEntity(type); showSidebar = false" draggable="true" @dragstart="handleDragStartSidebar($event, type)" @dragend="handleDragEnd"
               class="flex items-center gap-2 lg:gap-3 p-2 bg-slate-100 rounded-sm cursor-pointer border border-transparent hover:border-slate-300">
            <div :class="entityDefs[type].colorClass" class="w-5 h-5 lg:w-6 lg:h-6 flex items-center justify-center rounded-sm text-xs">{{ entityDefs[type].icon }}</div>
            <span class="text-xs lg:text-sm font-medium">{{ entityDefs[type].label }}</span>
          </div>
        </div>
      </div>
      <div>
        <h3 class="text-xs font-semibold text-slate-400 mb-2 uppercase tracking-wider">Others</h3>
        <div class="grid grid-cols-2 lg:grid-cols-1 gap-2">
          <div v-for="type in (['farm', 'statue', 'mountain'] as const)" :key="type"
               @click="placeEntity(type); showSidebar = false" draggable="true" @dragstart="handleDragStartSidebar($event, type)" @dragend="handleDragEnd"
               class="flex items-center gap-2 lg:gap-3 p-2 bg-slate-100 rounded-sm cursor-pointer border border-transparent hover:border-slate-300">
            <div :class="entityDefs[type].colorClass" class="w-5 h-5 lg:w-6 lg:h-6 flex items-center justify-center rounded-sm text-xs">{{ entityDefs[type].icon }}</div>
            <span class="text-xs lg:text-sm font-medium">{{ entityDefs[type].label }}</span>
          </div>
        </div>
      </div>
      <div class="mt-auto pt-4 border-t lg:border-none flex flex-col gap-2">
          <label class="flex items-center gap-2 p-2 bg-slate-100 rounded-sm cursor-pointer text-xs font-medium border border-transparent">
            <input type="checkbox" v-model="hideEffectArea" class="rounded-sm w-4 h-4 text-slate-700 focus:ring-slate-500 border-gray-300" />
            Hide Areas
          </label>
          <button v-if="isAdmin" @click="currentConfigId ? saveConfig() : showSaveModal = true" 
                  :disabled="isSaving"
                  class="w-full p-3 bg-slate-800 text-white rounded-sm text-xs font-bold uppercase tracking-widest active:scale-[0.98] transition-transform disabled:opacity-50">
            {{ isSaving ? '⌛ Saving...' : (currentConfigId ? '💾 Update Board' : '💾 Save & Share') }}
          </button>
          <button v-else @click="unlockAdmin" class="w-full p-3 bg-emerald-600 text-white rounded-sm text-xs font-bold uppercase tracking-widest active:scale-[0.98] transition-transform">
            🔑 Unlock Admin
          </button>
          <button v-if="currentConfigSlug" @click="showShareModal = true" class="w-full p-2 bg-slate-100 text-slate-600 rounded-sm text-[10px] font-bold uppercase border border-slate-200">
            🔗 View Share Links
          </button>
      </div>
    </aside>

    <!-- Save Modal -->
    <div v-if="showSaveModal" class="fixed inset-0 bg-slate-900/50 backdrop-blur-sm z-[100] flex items-center justify-center p-4">
      <div class="bg-white rounded-sm shadow-2xl w-full max-w-md p-6 border border-slate-200">
        <h2 class="text-lg font-bold mb-4 flex items-center gap-2">
          <span class="text-2xl">🔒</span> Protect Your Board
        </h2>
        <p class="text-sm text-slate-500 mb-6">Create a password to manage this board later. People with the link can only view it.</p>
        
        <div class="space-y-4">
          <div class="flex flex-col gap-1">
            <label class="text-[10px] font-bold uppercase text-slate-400">Admin Password</label>
            <input type="password" v-model="savePassword" class="w-full p-3 bg-slate-50 border border-slate-200 rounded-sm focus:border-slate-800 outline-none transition-colors" placeholder="••••••••" />
          </div>
          
          <div class="flex gap-2 pt-2">
            <button @click="showSaveModal = false" class="flex-1 p-3 text-xs font-bold uppercase tracking-wider text-slate-500 hover:bg-slate-50 rounded-sm italic">Cancel</button>
            <button @click="saveConfig" :disabled="!savePassword" class="flex-1 p-3 bg-slate-800 text-white text-xs font-bold uppercase tracking-wider rounded-sm disabled:opacity-50">Save Board</button>
          </div>
        </div>
      </div>
    </div>

    <!-- Share Modal -->
    <div v-if="showShareModal" class="fixed inset-0 bg-slate-900/50 backdrop-blur-sm z-[100] flex items-center justify-center p-4">
      <div class="bg-white rounded-sm shadow-2xl w-full max-w-md p-6 border border-slate-200">
        <h2 class="text-lg font-bold mb-4 flex items-center gap-2">
          <span class="text-2xl">🚀</span> Board Shared!
        </h2>
        <p class="text-sm text-slate-500 mb-6">Your board is live. Share the link below with others.</p>
        
        <div class="space-y-6">
          <div>
            <label class="text-[10px] font-bold uppercase text-slate-400 block mb-2">Shareable Link (Read-only)</label>
            <div class="flex gap-2">
              <input readonly :value="shareLink" class="flex-1 p-3 bg-slate-50 border border-slate-100 rounded-sm text-xs font-mono text-slate-600 outline-none" />
              <button @click="copyToClipboard(shareLink)" class="px-4 bg-slate-800 text-white rounded-sm text-xs font-bold flex items-center justify-center">Copy</button>
            </div>
          </div>

          <div class="p-4 bg-amber-50 border border-amber-100 rounded-sm">
            <p class="text-[10px] font-bold text-amber-700 uppercase mb-1">⚠️ Admin Access</p>
            <p class="text-xs text-amber-800">To edit this board later on another device, use the password you created. Don't share it!</p>
          </div>
          
          <button @click="showShareModal = false" class="w-full p-3 bg-slate-800 text-white text-xs font-bold uppercase tracking-wider rounded-sm">Done</button>
        </div>
      </div>
    </div>

    <main class="flex-1 flex flex-col bg-slate-200 overflow-hidden relative">
      


      <!-- Zoom Controls -->
      <div class="fixed bottom-24 right-4 flex flex-col gap-2 z-40">
         <button @click="zoomIn" class="w-10 h-10 bg-white border border-slate-200 shadow-sm rounded-sm flex items-center justify-center font-bold text-lg hover:bg-slate-50 transition-colors">+</button>
         <button @click="zoomOut" class="w-10 h-10 bg-white border border-slate-200 shadow-sm rounded-sm flex items-center justify-center font-bold text-lg hover:bg-slate-50 transition-colors">-</button>
      </div>

      <!-- Board Container (Top-down view) -->
      <div class="flex-1 w-full overflow-auto flex items-center justify-center p-12 lg:p-24 hide-scrollbar scroll-smooth">
          <div class="relative transition-transform duration-300 ease-out"
               :style="{ transform: `scale(${viewportScale})` }">
               
               <div class="grid relative bg-white border border-slate-300 select-none touch-none shadow-[15px_15px_30px_rgba(0,0,0,0.05)]"
                    :style="{ gridTemplateColumns: `repeat(${boardWidth}, minmax(0, 1fr))`, gridTemplateRows: `repeat(${boardHeight}, minmax(0, 1fr))`, width: '1000px', height: '1000px', flexShrink: 0 }">
                
                <!-- Grid Background -->
                <div v-for="index in boardWidth * boardHeight" :key="`cell-${index}`"
                    @dragover="handleDragOver($event, (index - 1) % boardWidth, Math.floor((index - 1) / boardWidth))" 
                    @drop="handleDrop($event, (index - 1) % boardWidth, Math.floor((index - 1) / boardWidth))"
                    @click="clickGridCell((index - 1) % boardWidth, Math.floor((index - 1) / boardWidth))"
                    class="border-r border-b border-slate-50 bg-transparent relative box-border hover:bg-slate-50/50 transition-colors"
                    :class="{'border-l border-t': index === 1, 'border-l-0': (index - 1) % boardWidth !== 0, 'border-t-0': Math.floor((index - 1) / boardWidth) !== 0}">
                </div>

                <!-- Preview Layer -->
                <div v-if="dragX !== null && dragY !== null && dragType"
                    class="absolute pointer-events-none opacity-50 z-20 shadow-md"
                    :class="[entityDefs[dragType].class, isSpaceFree(dragX, dragY, entityDefs[dragType].w, entityDefs[dragType].h, dragMoveId || undefined) ? 'bg-emerald-300' : 'bg-rose-300']"
                    :style="{ 
                        left: `${(dragX / boardWidth) * 100}%`, 
                        top: `${(dragY / boardHeight) * 100}%`, 
                        width: `${(entityDefs[dragType].w / boardWidth) * 100}%`, 
                        height: `${(entityDefs[dragType].h / boardHeight) * 100}%` 
                    }">
                </div>

                <div v-if="!hideEffectArea" class="absolute inset-0 w-full h-full pointer-events-none z-0">
                    <template v-for="ent in entities" :key="'effect-'+ent.id">
                        <div v-if="ent.type === 'flag'" class="absolute bg-blue-400/10 border border-blue-400/20"
                            :style="{ 
                                left: `${(ent.x - 3) / boardWidth * 100}%`, 
                                top: `${(ent.y - 3) / boardHeight * 100}%`, 
                                width: `${7 / boardWidth * 100}%`, 
                                height: `${7 / boardHeight * 100}%` 
                            }">
                        </div>
                    </template>
                </div>

                <!-- Entities -->
                <div v-for="ent in entities" :key="ent.id" 
                    class="absolute p-[1px] transition-all transform-gpu pointer-events-none"
                    :style="{ 
                        left: `${(ent.x / boardWidth) * 100}%`, 
                        top: `${(ent.y / boardHeight) * 100}%`, 
                        width: `${(entityDefs[ent.type].w / boardWidth) * 100}%`, 
                        height: `${(entityDefs[ent.type].h / boardHeight) * 100}%`, 
                        opacity: ((listHoveredEntityId || selectedEntityId) && ent.id !== (listHoveredEntityId || selectedEntityId)) ? 0.2 : 1,
                        zIndex: (selectedEntityId === ent.id || hoveredEntityId === ent.id) ? 1000 : 10
                    }">
                    
                    <div draggable="true" 
                            @dragstart="handleDragStartBoard($event, ent.id)"
                            @dragend="handleDragEnd"
                            @click.stop="selectEntity(ent.id)"
                             @mouseenter="hoveredEntityId = ent.id"
                             @mouseleave="hoveredEntityId = null"
                             @dblclick.stop="ent.cityNumber ? (editingEntityId = ent.id) : null"
                             :class="[entityDefs[ent.type].class, 'w-full h-full flex flex-col items-center justify-center cursor-move pointer-events-auto border-2 transition-transform active:scale-95 relative group', selectedEntityId === ent.id ? 'border-emerald-500 scale-105 z-50' : 'border-transparent']">
                        
                        <!-- Entity Tooltip -->
                        <div v-if="getHoveredEntity?.id === ent.id && editingEntityId !== ent.id" 
                             class="absolute bottom-[calc(100%+8px)] left-1/2 -translate-x-1/2 z-[1100] pointer-events-none">
                            <div class="bg-slate-800 text-white px-3 py-1.5 rounded-sm shadow-2xl flex items-center gap-2 border border-slate-700 whitespace-nowrap">
                                <span class="text-[10px] font-bold uppercase tracking-widest">{{ ent.name || entityDefs[ent.type].label }}</span>
                            </div>
                            <!-- Tooltip Arrow -->
                            <div class="w-2 h-2 bg-slate-800 border-r border-b border-slate-700 rotate-45 mx-auto -mt-1"></div>
                        </div>

                        <!-- Content stay flat-ish but recognizable -->
                        <div class="flex flex-col items-center justify-center transform-gpu w-full px-1">
                            <span v-if="entityDefs[ent.type].icon" :class="[entityDefs[ent.type].w >= 2 ? 'text-2xl' : 'text-xs']">{{ entityDefs[ent.type].icon }}</span>
                            
                            <template v-if="ent.cityNumber">
                                <input v-if="editingEntityId === ent.id" 
                                       v-model="ent.name" 
                                       @blur="editingEntityId = null"
                                       @keyup.enter="editingEntityId = null"
                                       class="w-full text-[10px] font-bold text-center bg-white/90 rounded-sm border-none focus:ring-1 focus:ring-slate-800 py-0.5 mt-1"
                                       autofocus
                                       @click.stop />
                                <span v-else class="text-[10px] font-bold text-slate-800/80 truncate w-full text-center mt-0.5 leading-none">{{ ent.name }}</span>
                                <span class="font-bold text-slate-800/40 text-[9px] mt-0.5">#{{ ent.cityNumber }}</span>
                            </template>
                        </div>
                    </div>
                </div>
              </div>
          </div>
      </div>

      <!-- Editor Panel -->
      <transition enter-active-class="transition duration-200 ease-out" enter-from-class="translate-y-10 opacity-0" enter-to-class="translate-y-0 opacity-100" leave-active-class="transition duration-150 ease-in" leave-from-class="translate-y-0 opacity-100" leave-to-class="translate-y-10 opacity-0">
        <div v-if="getSelectedEntity" class="fixed bottom-4 left-1/2 -track-x-1/2 w-[90%] lg:w-full lg:max-w-sm bg-white border border-slate-200 rounded-sm p-3 shadow-2xl flex items-center gap-4 z-50 transform -translate-x-1/2">
           <div class="flex items-center gap-3 w-full">
             <div :class="entityDefs[getSelectedEntity.type].class" class="w-10 h-10 flex-shrink-0 relative flex items-center justify-center rounded-sm">
                 <span v-if="entityDefs[getSelectedEntity.type].icon" class="text-lg">{{ entityDefs[getSelectedEntity.type].icon }}</span>
                 <span v-if="getSelectedEntity.cityNumber" class="font-bold text-slate-800/60 text-[10px] absolute bottom-0 right-0.5">{{ getSelectedEntity.cityNumber }}</span>
             </div>
             <div class="flex-1">
               <input v-if="getSelectedEntity.name !== undefined" type="text" v-model="getSelectedEntity.name" class="w-full text-sm font-bold text-slate-800 bg-transparent border-b border-slate-200 focus:border-slate-800 focus:outline-none transition-colors" placeholder="City Name..." maxlength="16"/>
               <p v-else class="text-[10px] font-bold text-slate-500 uppercase tracking-widest">{{ entityDefs[getSelectedEntity.type].label }}</p>
             </div>
           </div>
           <button @click="deleteEntity(getSelectedEntity.id); selectedEntityId = null" class="ml-auto p-2.5 text-rose-500 bg-rose-50 rounded-sm hover:bg-rose-100 transition-colors">
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/></svg>
           </button>
        </div>
      </transition>
    </main>

    <!-- Cities List (Mobile Drawer) -->
    <aside :class="['w-full lg:w-64 bg-white border-l border-slate-200 flex flex-col overflow-y-auto lg:shrink-0 transition-all duration-300 z-20 shadow-none', 
                    showCitiesList ? 'fixed inset-x-0 top-[65px] bottom-0 lg:static' : 'hidden lg:flex']">
      <div class="p-4 border-b border-slate-100 sticky top-0 bg-white/90 backdrop-blur z-20 flex justify-between items-center">
        <h3 class="text-xs font-semibold text-slate-400 uppercase tracking-wider">Strategic Cities</h3>
      </div>
      <div class="flex flex-col gap-1 p-3 overflow-y-auto w-full">
        <div v-for="city in citiesList" :key="city.id" 
             @mouseenter="listHoveredEntityId = city.id" 
             @mouseleave="listHoveredEntityId = null" 
             @click="selectEntity(city.id); showCitiesList = false" 
             class="p-2 border rounded-sm cursor-pointer transition-all flex items-center gap-3" 
             :class="{'border-emerald-500 bg-emerald-50': selectedEntityId === city.id, 'border-slate-200 bg-slate-50': listHoveredEntityId === city.id && selectedEntityId !== city.id, 'border-slate-100 bg-white': selectedEntityId !== city.id && listHoveredEntityId !== city.id }">
          <div :class="entityDefs[city.type].colorClass" class="w-6 h-6 rounded-sm flex-shrink-0 flex items-center justify-center font-bold text-slate-800/60 text-xs shadow-sm">{{ city.cityNumber }}</div>
          <span class="text-xs font-semibold text-slate-700 truncate flex-1 uppercase tracking-tight">{{ city.name }}</span>
        </div>
        <div v-if="citiesList.length === 0" class="text-[10px] text-slate-400 text-center py-10 border-dashed border border-slate-200 rounded-sm italic uppercase tracking-widest">No deployments available</div>
      </div>
    </aside>
  </div>
</template>

<style>
.hide-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
.hide-scrollbar::-webkit-scrollbar { display: none; }



input[type=range]::-webkit-slider-thumb {
  height: 14px;
  width: 14px;
  border-radius: 2px;
  background: #1e293b;
  cursor: pointer;
  -webkit-appearance: none;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
}

.grid > div {
    backface-visibility: hidden;
}
</style>
