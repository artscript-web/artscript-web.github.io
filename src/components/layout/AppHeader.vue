<!-- src/components/layout/AppHeader.vue - ENHANCED -->
<template>
  <div class="toolbar-container">
    <header class="toolbar">
      <div class="toolbar-left">
        <!-- File Menu -->
        <div class="dropdown" :class="{ active: isFileMenuOpen }" @click.stop="toggleFileMenu">
          <button class="icon-btn dropdown-btn" title="File Menu" @click.stop="toggleFileMenu">
            <span class="hamburger-icon"> <span></span><span></span><span></span> </span>
          </button>

          <div class="dropdown-menu" v-if="isFileMenuOpen">
            <div class="dropdown-item-with-submenu">
              <button class="dropdown-item" @click.stop><i class="pi pi-plus"></i> New <i class="pi pi-angle-right" style="float: right; margin-top: 2px;"></i></button>
              <div class="dropdown-submenu">
                <button class="dropdown-item" @click="createNewProject('Film')">
                  <i class="pi pi-video"></i> Film
                </button>
                <button class="dropdown-item" @click="createNewProject('TV Show')">
                  <i class="pi pi-desktop"></i> TV Show
                </button>
                <button
                  class="dropdown-item dropdown-item-coming-soon"
                  title="Book format coming soon"
                  @click="showBookComingSoon = true; isFileMenuOpen = false"
                >
                  <i class="pi pi-book"></i> Book
                </button>
              </div>
            </div>
            <button class="dropdown-item" @click="emitOpen">
              <i class="pi pi-folder-open"></i> Open
            </button>
            <button class="dropdown-item" @click="store.exportProjectAsJSON()">
              <i class="pi pi-save"></i> Save
            </button>

            <div class="dropdown-divider"></div>

            <button class="dropdown-item" @click="store.showTitleDialog = true">
              <i class="pi pi-file"></i> Title Page
            </button>

            <div class="dropdown-divider"></div>

            <button class="dropdown-item" @click="store.showPreviewDialog = true">
              <i class="pi pi-eye"></i> Preview
            </button>
            <button class="dropdown-item" @click="store.showPDFDialog = true">
              <i class="pi pi-file-pdf"></i> Export PDF
            </button>
            <button class="dropdown-item" @click="store.exportToFountain()">
              <i class="pi pi-download"></i> Export Fountain
            </button>

            <div class="dropdown-divider"></div>

            <!-- Force Element Type -->
            <div class="dropdown-item-with-submenu">
              <button class="dropdown-item" @click.stop>
                <i class="pi pi-bolt"></i> Force.. 
                <i class="pi pi-angle-right" style="float: right; margin-top: 2px;"></i>
              </button>
              <div class="dropdown-submenu">
                <button class="dropdown-item" @click="forceLineType('scene-heading')">
                  <span>Force Scene Heading</span>
                  <span class="shortcut-hint">Ctrl+1</span>
                </button>
                <button class="dropdown-item" @click="forceLineType('action')">
                  <span>Force Action</span>
                  <span class="shortcut-hint">Ctrl+2</span>
                </button>
                <button class="dropdown-item" @click="forceLineType('character')">
                  <span>Force Character</span>
                  <span class="shortcut-hint">Ctrl+3</span>
                </button>
                <button class="dropdown-item" @click="forceLineType('dialogue')">
                  <span>Force Dialogue</span>
                  <span class="shortcut-hint">Ctrl+4</span>
                </button>
                <button class="dropdown-item" @click="forceLineType('transition')">
                  <span>Force Transition</span>
                  <span class="shortcut-hint">Ctrl+5</span>
                </button>
              </div>
            </div>

            <!-- Tools submenu -->
            <div class="dropdown-item-with-submenu">
              <button class="dropdown-item" @click.stop>
                <i class="pi pi-cog"></i> Tools
                <i class="pi pi-angle-right" style="float: right; margin-top: 2px;"></i>
              </button>
              <div class="dropdown-submenu">
                <button class="dropdown-item" @click="store.showTimeReminder = true; isFileMenuOpen = false">
                  <i class="pi pi-clock"></i> Time Reminder
                </button>
                <button class="dropdown-item" @click="openFindReplace('find'); isFileMenuOpen = false">
                  <i class="pi pi-search"></i> Find (Ctrl+F)
                </button>
                <button class="dropdown-item" @click="openFindReplace('replace'); isFileMenuOpen = false">
                  <i class="pi pi-replay"></i> Replace (Ctrl+H)
                </button>
                <button class="dropdown-item" @click="store.showCharacterReplace = true; isFileMenuOpen = false">
                  <i class="pi pi-user-edit"></i> Replace Character
                </button>
                <button class="dropdown-item" @click="store.showScriptAnalysis = true; isFileMenuOpen = false">
                  <i class="pi pi-chart-bar"></i> Script Analysis
                </button>
                <button class="dropdown-item" @click="store.showSpellGrammarDialog = true; isFileMenuOpen = false">
                  <i class="pi pi-check-circle"></i> Spell & Grammar
                </button>
              </div>
            </div>

            <div class="dropdown-divider"></div>

            <button class="dropdown-item" @click="store.showTrainingDialog = true; isFileMenuOpen = false">
              <i class="pi pi-book"></i> Training
            </button>
            <button class="dropdown-item" @click="showShortcuts">
              <i class="pi pi-question-circle"></i> Keyboard Shortcuts (Ctrl+/)
            </button>
          </div>
        </div>

        <!-- Scene Navigator Toggle -->
        <button
          class="icon-btn"
          :class="{ active: store.sceneNavVisible }"
          title="Toggle Scene Navigator (Ctrl+\)"
          @click="store.sceneNavVisible = !store.sceneNavVisible"
        >
          <span class="file-icon">
            <span class="file-icon-lines"></span>
            <span class="file-icon-lines"></span>
            <span class="file-icon-lines"></span>
          </span>
        </button>

        <!-- Spell check toggle (native browser spellcheck) -->
        <button
          class="icon-btn"
          :class="{ active: store.spellCheckEnabled }"
          :title="store.spellCheckEnabled ? 'Spell check on' : 'Spell check off'"
          @click="store.spellCheckEnabled = !store.spellCheckEnabled"
        >
          <i class="pi pi-check" style="font-size: 0.9rem;"></i>
        </button>

        <h1 class="app-title-inline clickable-title" @click="$emit('open-about')">ArtScript Web</h1>

        <!-- Undo/Redo (NEW) -->
        <div class="undo-redo-group">
          <button
            class="icon-btn"
            :disabled="!store.canUndo()"
            title="Undo (Ctrl+Z)"
            @click="store.undo()"
          >
            <i class="pi pi-undo"></i>
          </button>
          <button
            class="icon-btn"
            :disabled="!store.canRedo()"
            title="Redo (Ctrl+Shift+Z)"
            @click="store.redo()"
          >
            <i class="pi pi-redo"></i>
          </button>
        </div>
      </div>

      <div class="toolbar-center">
        <input
          v-if="store.activeProject"
          type="text"
          class="project-name-input"
          v-model="store.activeProject.name"
          spellcheck="false"
        />
      </div>

      <div class="toolbar-right">
        <!-- Stats Display -->
        <div v-if="stats" class="stats-display">
          <span class="stat-item" title="Page count">
            <i class="pi pi-file"></i> {{ stats.pageCount }}
          </span>
          <span v-if="daysRemaining !== null" class="stat-item" :title="reminderTooltip">
            <i class="pi pi-clock"></i> {{ daysRemainingText }}
          </span>
        </div>

        <!-- Full Page View Toggle -->
        <button
          class="icon-btn"
          :class="{ active: store.fullPageView }"
          title="Toggle Full Page View"
          @click="store.toggleFullPageView"
        >
          <i class="pi pi-eye"></i>
        </button>

        <!-- Dark Mode Toggle -->
        <label class="switch">
          <input type="checkbox" :checked="store.darkMode" @change="store.toggleDarkMode" />
          <span class="slider"></span>
        </label>

        <!-- Format Display -->
        <div class="info-text">
          <span v-if="store.activeProject">{{ store.activeProject.format }}</span>
        </div>
      </div>
    </header>

    <TabBar />

    <!-- Book format coming soon popup -->
    <div v-if="showBookComingSoon" class="book-coming-soon-overlay" @click.self="showBookComingSoon = false">
      <div class="book-coming-soon-dialog">
        <h2 class="book-coming-soon-title">ArtScript <strong>Book</strong></h2>
        <p class="book-coming-soon-text">
          Book format is coming soon. You will be able to write long-form prose with chapters and body text. For now, use Film or TV Show for screenplay projects.
        </p>
        <button type="button" class="book-coming-soon-close" @click="showBookComingSoon = false">OK</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, onMounted, onUnmounted } from 'vue'
import { useProjectStore } from '@/stores/project'
import { useRouter } from 'vue-router'
import TabBar from './TabBar.vue'

const store = useProjectStore()
const router = useRouter()

const isFileMenuOpen = ref(false)
const showBookComingSoon = ref(false)

const stats = computed(() => store.scriptStats)

// Calculate days remaining until deadline
const daysRemaining = computed(() => {
  const deadline = store.activeProject?.deadline
  if (!deadline || !deadline.date) return null
  
  const today = new Date()
  today.setHours(0, 0, 0, 0)
  
  const deadlineDate = new Date(deadline.date)
  deadlineDate.setHours(0, 0, 0, 0)
  
  // Only show if deadline is in the future
  if (deadlineDate <= today) return null
  
  const diffTime = deadlineDate - today
  const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24))
  
  return diffDays
})

const daysRemainingText = computed(() => {
  if (daysRemaining.value === null) return ''
  if (daysRemaining.value === 0) return 'Today'
  if (daysRemaining.value === 1) return '1 day'
  return `${daysRemaining.value} days`
})

const reminderTooltip = computed(() => {
  const deadline = store.activeProject?.deadline
  if (!deadline || !deadline.date) return ''
  
  const date = new Date(deadline.date)
  const formattedDate = date.toLocaleDateString('en-US', { 
    year: 'numeric', 
    month: 'long', 
    day: 'numeric' 
  })
  
  let tooltip = `Deadline: ${formattedDate}`
  if (deadline.time) {
    tooltip += ` at ${deadline.time}`
  }
  if (deadline.note) {
    tooltip += `\n${deadline.note}`
  }
  
  return tooltip
})

const toggleFileMenu = () => {
  isFileMenuOpen.value = !isFileMenuOpen.value
}

// Close menu when clicking outside
const closeMenuOnOutsideClick = (event) => {
  if (isFileMenuOpen.value && !event.target.closest('.dropdown')) {
    isFileMenuOpen.value = false
  }
}

const emitNew = () => {
  router.push('/')
  isFileMenuOpen.value = false
}

const createNewProject = (format) => {
  // Close menu immediately
  isFileMenuOpen.value = false
  
  // Use nextTick to ensure menu closes before navigation
  nextTick(() => {
    // Create project and navigate
    const id = store.createProject(format)
    store.saveToRecentProjects(id)
    router.push(`/project/${id}`)
  })
}

const emitOpen = () => {
  document.getElementById('hidden-file-input')?.click()
  isFileMenuOpen.value = false
}

const openFindReplace = (mode) => {
  store.showFindReplace = true
  window.dispatchEvent(new CustomEvent('open-find-replace', { detail: { mode } }))
  isFileMenuOpen.value = false
}

const showShortcuts = () => {
  window.dispatchEvent(new CustomEvent('show-shortcuts-dialog'))
  isFileMenuOpen.value = false
}

const forceLineType = (type) => {
  // Dispatch event to force line type in editor
  window.dispatchEvent(new CustomEvent('force-line-type', { detail: { type } }))
  isFileMenuOpen.value = false
}

// Setup click outside handler
onMounted(() => {
  document.addEventListener('click', closeMenuOnOutsideClick)
})

onUnmounted(() => {
  document.removeEventListener('click', closeMenuOnOutsideClick)
})
</script>

<style scoped>
/* Previous styles remain... */

.undo-redo-group {
  display: flex;
  gap: 4px;
  margin-left: 15px;
  padding-left: 15px;
  border-left: 1px solid #e0e0e0;
}

:global(body.dark-mode) .undo-redo-group {
  border-left-color: #444;
}

.icon-btn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.icon-btn.active {
  background: rgba(0, 0, 0, 0.08);
  border-color: #b0b0b0;
}

:global(body.dark-mode) .icon-btn.active {
  background: rgba(255, 255, 255, 0.15);
  border-color: #666;
}

.stats-display {
  display: flex;
  gap: 15px;
  font-size: 13px;
  color: #666;
  padding-right: 15px;
  border-right: 1px solid #e0e0e0;
}

:global(body.dark-mode) .stats-display {
  color: #aaa;
  border-right-color: #444;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: help;
}

.stat-item i {
  font-size: 12px;
}

.dropdown-item {
  display: flex;
  align-items: center;
  gap: 10px;
}

.dropdown-item i {
  width: 16px;
  font-size: 14px;
  opacity: 0.7;
}

.shortcut-hint {
  color: #999;
  font-size: 12px;
  margin-left: auto;
  padding-left: 16px;
  font-family: 'Monaco', 'Menlo', 'Courier New', monospace;
  opacity: 0.7;
}

:global(body.dark-mode) .shortcut-hint {
  color: #666;
}

.dropdown-submenu .dropdown-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

/* Book option: greyed out, coming soon */
.dropdown-item-coming-soon {
  color: #999 !important;
  cursor: pointer;
  opacity: 0.85;
}
.dropdown-item-coming-soon:hover {
  background: rgba(0, 0, 0, 0.04);
  color: #666 !important;
}
:global(body.dark-mode) .dropdown-item-coming-soon {
  color: #666 !important;
}
:global(body.dark-mode) .dropdown-item-coming-soon:hover {
  color: #888 !important;
  background: rgba(255, 255, 255, 0.06);
}

/* Book coming soon popup */
.book-coming-soon-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10000;
}
.book-coming-soon-dialog {
  background: white;
  border-radius: 12px;
  padding: 24px 28px;
  max-width: 360px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
}
.book-coming-soon-title {
  margin: 0 0 12px 0;
  font-size: 20px;
  font-weight: 300;
  color: #333;
  font-family: 'Montserrat', sans-serif;
}
.book-coming-soon-title strong {
  font-weight: 900;
  -webkit-text-stroke: 0.4px currentColor;
  paint-order: stroke fill;
}
.book-coming-soon-text {
  margin: 0 0 20px 0;
  font-size: 14px;
  line-height: 1.5;
  color: #555;
}
.book-coming-soon-close {
  display: block;
  width: 100%;
  padding: 10px 16px;
  font-size: 14px;
  background: #333;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}
.book-coming-soon-close:hover {
  background: #444;
}
:global(body.dark-mode) .book-coming-soon-dialog {
  background: #2a2a2a;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5);
}
:global(body.dark-mode) .book-coming-soon-title {
  color: #e0e0e0;
}
:global(body.dark-mode) .book-coming-soon-text {
  color: #b0b0b0;
}
:global(body.dark-mode) .book-coming-soon-close {
  background: #444;
  color: #e0e0e0;
}
:global(body.dark-mode) .book-coming-soon-close:hover {
  background: #555;
}
</style>
