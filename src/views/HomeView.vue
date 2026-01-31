<template>
  <div class="launch-menu-overlay" :style="{ '--scale': scale }">
    <!-- Title: independent and centered initially; moves into left container when slide starts -->
    <div
      class="title-wrapper"
      :class="{
        'title-centered': !titleSlid,
        'title-left': titleSlid,
        'title-visible': titleVisible
      }"
    >
      <h1 class="launch-menu-title">
        <span class="typed-text">{{ displayedText }}</span>
        <span class="cursor-marker"></span>
      </h1>
      <p
        class="title-credit"
        :class="{ 'visible': showCredit }"
      >
        Created by Nick Christod
      </p>
    </div>

    <!-- Containers: visible when title appears; left container clips title once it slides in -->
    <div class="launch-menu-container" :class="{ 'visible': containersVisible }">
      <!-- Left Container: overflow hidden when title slides in -->
      <div class="launch-menu-left" :class="{ 'clip-title': titleSlid }">
      </div>

      <!-- Right Container: Actions -->
      <div class="launch-menu-right">
        <div class="launch-menu-section">
          <h2>Create New Project</h2>
          <div class="launch-menu-formats">
            <button class="launch-format-btn" @click="start('Film')">
              <i class="pi pi-video format-icon"></i>
              <span class="format-name">Film</span>
            </button>
            <button class="launch-format-btn" @click="start('TV Show')">
              <i class="pi pi-desktop format-icon"></i>
              <span class="format-name">TV Show</span>
            </button>
            <button class="launch-format-btn format-disabled" disabled>
              <i class="pi pi-book format-icon"></i>
              <span class="format-name">Book</span>
              <span class="coming-soon">Coming soon</span>
            </button>
          </div>
        </div>

        <div class="launch-menu-section">
          <input
            type="file"
            ref="fileInput"
            @change="handleImport"
            style="display: none"
            accept=".artsc,.json,.fountain"
          />
          <button class="open-file-btn" @click="$refs.fileInput.click()">
            <i class="pi pi-folder-open"></i> Open From File
          </button>
        </div>

        <div class="launch-menu-section">
          <h2>Recent Projects</h2>
          <div class="recent-projects-list">
            <div
              v-for="proj in recentProjects"
              :key="proj.id"
              class="recent-project-item"
              @click="openRecent(proj.id)"
            >
              {{ proj.name }} ({{ proj.format }})
            </div>

            <p v-if="recentProjects.length === 0" class="no-recent-projects">
              No recent projects found
            </p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { useRouter } from 'vue-router'
import { useProjectStore } from '@/stores/project'

const router = useRouter()
const store = useProjectStore()
const recentProjects = ref([])
const fileInput = ref(null)

const fullText = 'ArtScript Web'
const displayedText = ref('')
const isTyping = ref(true)
const titleVisible = ref(false)
const titleSlid = ref(false)
const containersVisible = ref(false)
const showCredit = ref(false)

// Unified scaling: base design is 1440x900, scale proportionally
const BASE_WIDTH = 1440
const BASE_HEIGHT = 900
const scale = ref(1)

const updateScale = () => {
  const scaleX = window.innerWidth / BASE_WIDTH
  const scaleY = window.innerHeight / BASE_HEIGHT
  // Use the smaller scale to ensure everything fits
  scale.value = Math.min(scaleX, scaleY, 1.2) // Cap at 1.2 to avoid giant sizes
}

onMounted(() => {
  recentProjects.value = store.loadRecentProjects()
  
  // Set initial scale
  updateScale()
  window.addEventListener('resize', updateScale)
  
  // Show title first
  setTimeout(() => {
    titleVisible.value = true
    startTypingAnimation()
  }, 100)
})

onUnmounted(() => {
  window.removeEventListener('resize', updateScale)
})

const startTypingAnimation = () => {
  let index = 0
  const typingInterval = setInterval(() => {
    if (index < fullText.length) {
      displayedText.value = fullText.substring(0, index + 1)
      index++
    } else {
      clearInterval(typingInterval)
      // Wait a bit before starting the slide animation
      setTimeout(() => {
        isTyping.value = false
        // Wait for cursor to fade, then slide title to left
        setTimeout(() => {
          // Reveal containers when slide starts
          containersVisible.value = true
          titleSlid.value = true
          // After title slides, reveal credit text
          setTimeout(() => {
            showCredit.value = true
          }, 600)
        }, 400)
      }, 800)
    }
  }, 100) // Typing speed: 100ms per character
}

const start = (format) => {
  const id = store.createProject(format)
  store.saveToRecentProjects(id)
  router.push(`/project/${id}`)
}

const openRecent = (id) => {
  // In a real app with backend, we would fetch the project data.
  // Since this is memory-based, we check if it's already in the store.
  const exists = store.projects.find((p) => p.id === id)
  if (exists) {
    store.activeProjectId = id
    router.push(`/project/${id}`)
  } else {
    alert(
      "Project data not in memory (Persistence requires Backend/File API). Please use 'Open From File'.",
    )
  }
}

const handleImport = async (e) => {
  const file = e.target.files[0]
  if (!file) return
  const text = await file.text()
  const fileName = file.name.toLowerCase()
  
  let id
  if (fileName.endsWith('.fountain')) {
    id = store.importProjectFromFountain(text, file.name)
  } else {
    id = store.importProjectFromJSON(text, file.name)
  }
  
  if (id) {
    store.saveToRecentProjects(id)
    router.push(`/project/${id}`)
  }
}
</script>

<style scoped>
/* Unified scaling using CSS custom property */
:deep(.launch-menu-overlay) {
  --scale: 1;
}

/* Left container: overflow hidden when title slides in to keep it bounded */
:deep(.launch-menu-left) {
  overflow: visible;
  display: flex;
  align-items: center;
  justify-content: flex-start;
  padding: calc(60px * var(--scale));
  position: relative;
}

:deep(.launch-menu-left.clip-title) {
  overflow: hidden;
}

/* Title: starts fixed and centered on viewport, then slides into left container */
:deep(.title-wrapper) {
  position: fixed;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  opacity: 0;
  transition: all 1.2s cubic-bezier(0.4, 0, 0.2, 1);
  z-index: 100;
}

:deep(.title-wrapper.title-visible) {
  opacity: 1;
}

/* Centered: middle of viewport */
:deep(.title-wrapper.title-centered) {
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) scale(var(--scale));
}

/* Slid: moves into left container position and stays clipped there */
:deep(.title-wrapper.title-left) {
  top: 50%;
  left: calc(5% + 60px * var(--scale));
  transform: translateY(-50%) scale(var(--scale));
  transform-origin: left center;
}

/* Scale the title text */
:deep(.launch-menu-title) {
  font-size: calc(64px * var(--scale));
}

:deep(.title-credit) {
  font-size: calc(14px * var(--scale));
  margin-top: calc(12px * var(--scale));
}

/* Scale the container content */
:deep(.launch-menu-container) {
  --scale: inherit;
}

:deep(.launch-menu-right) {
  padding: calc(60px * var(--scale));
}

:deep(.launch-menu-section) {
  margin-bottom: calc(40px * var(--scale));
}

:deep(.launch-menu-section h2) {
  font-size: calc(16px * var(--scale));
  margin-bottom: calc(20px * var(--scale));
}

:deep(.launch-menu-formats) {
  gap: calc(20px * var(--scale));
}

:deep(.launch-format-btn) {
  padding: calc(20px * var(--scale)) calc(16px * var(--scale));
  gap: calc(8px * var(--scale));
  border-radius: calc(4px * var(--scale));
}

:deep(.format-icon) {
  font-size: calc(20px * var(--scale));
}

:deep(.format-name) {
  font-size: calc(18px * var(--scale));
}

:deep(.coming-soon) {
  font-size: calc(11px * var(--scale));
}

:deep(.open-file-btn) {
  padding: calc(14px * var(--scale)) calc(20px * var(--scale));
  font-size: calc(14px * var(--scale));
  gap: calc(8px * var(--scale));
  border-radius: calc(4px * var(--scale));
}

:deep(.recent-project-item) {
  padding: calc(14px * var(--scale)) calc(16px * var(--scale));
  margin-bottom: calc(8px * var(--scale));
  font-size: calc(14px * var(--scale));
  border-radius: calc(4px * var(--scale));
}

:deep(.no-recent-projects) {
  font-size: calc(14px * var(--scale));
  padding: calc(20px * var(--scale));
}

:deep(.recent-projects-list) {
  max-height: calc(180px * var(--scale));
}
</style>
