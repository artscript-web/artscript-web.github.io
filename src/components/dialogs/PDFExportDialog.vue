<!-- PDFExportDialog.vue - REPLACE YOUR EXISTING ONE -->
<template>
  <Dialog
    v-model:visible="visible"
    header="Export PDF"
    :modal="true"
    :style="{ width: '600px' }"
    @hide="resetOptions"
    :dismissableMask="true"
  >
    <div class="export-options">
      <div class="option-field">
        <label>Filename</label>
        <InputText v-model="filename" placeholder="my-script.pdf" class="w-full" />
      </div>

      <div class="option-field">
        <label>Include Title Page</label>
        <div class="checkbox-wrapper">
          <input type="checkbox" id="includeTitlePage" v-model="includeTitlePage" />
          <label for="includeTitlePage">Add title page to PDF</label>
        </div>
      </div>

      <div class="option-field">
        <label>Scene Numbers</label>
        <div class="checkbox-wrapper">
          <input type="checkbox" id="includeSceneNumbers" v-model="includeSceneNumbers" />
          <label for="includeSceneNumbers">Show scene numbers</label>
        </div>
      </div>

      <div class="option-field">
        <label>Print Notes</label>
        <div class="checkbox-wrapper">
          <input type="checkbox" id="printNotes" v-model="printNotes" />
          <label for="printNotes">Include notes in PDF export</label>
        </div>
        <small class="option-explanation">Notes (lines starting with "//") will appear in italic font if enabled</small>
      </div>

      <div class="option-field">
        <label>Language</label>
        <select v-model="language" class="w-full language-select">
          <option value="en">English</option>
          <option value="el">Greek (Ελληνικά)</option>
        </select>
      </div>

      <div v-if="isExporting" class="export-progress">
        <div class="progress-spinner"></div>
        <p>Generating PDF...</p>
      </div>
    </div>

    <template #footer>
      <Button
        label="Cancel"
        severity="secondary"
        text
        @click="visible = false"
        :disabled="isExporting"
      />
      <Button label="Export PDF" icon="pi pi-file-pdf" @click="exportPDF" :loading="isExporting" />
    </template>
  </Dialog>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { useProjectStore } from '@/stores/project'
import Dialog from 'primevue/dialog'
import Button from 'primevue/button'
import InputText from 'primevue/inputtext'
import { jsPDF } from 'jspdf'

const props = defineProps(['visible'])
const emit = defineEmits(['update:visible'])

const store = useProjectStore()
const filename = ref('')
const includeTitlePage = ref(true)
const includeSceneNumbers = ref(true)
const printNotes = ref(false)
const language = ref('en')
const isExporting = ref(false)

const visible = computed({
  get: () => props.visible,
  set: (val) => emit('update:visible', val),
})

// Initialize filename from project name
watch(
  () => props.visible,
  (newVal) => {
    if (newVal && store.activeProject) {
      filename.value = store.activeProject.name.replace(/\s+/g, '_') + '.pdf'
    }
  },
)

const resetOptions = () => {
  includeTitlePage.value = true
  includeSceneNumbers.value = true
  printNotes.value = false
  language.value = 'en'
  isExporting.value = false
}

const exportPDF = async () => {
  if (!store.activeProject) return

  isExporting.value = true

  try {
    // Create PDF in portrait letter size
    const pdf = new jsPDF({
      orientation: 'portrait',
      unit: 'pt',
      format: 'letter', // 8.5" x 11" = 612 x 792 pts
      compress: true,
    })
    
    // Enable UTF-8 encoding for Greek characters
    pdf.setLanguage(language.value === 'el' ? 'el' : 'en')

    // Prevent extra space between letters (fixes sporadic spacing on some browsers/environments e.g. GitHub)
    pdf.setCharSpace(0)

    const pageWidth = 612
    const pageHeight = 792
    const leftMargin = 108 // 1.5 inches
    const rightMargin = 72 // 1 inch
    const topMargin = 40 // ~0.56 inches (reduced to allow 1 more line at top)
    const bottomMargin = 5 // ~0.07 inches (minimal margin, allows 2 more lines at bottom)
    const usableWidth = pageWidth - leftMargin - rightMargin

    let yPosition = topMargin
    let pageNumber = 1

    // Set font
    pdf.setFont('courier')

    // Add title page if requested
    if (includeTitlePage.value && store.activeProject.titlePage) {
      addTitlePage(pdf, pageWidth, pageHeight)
      pdf.addPage()
      pageNumber++
      pdf.setCharSpace(0) // Reset so first script line doesn't get wrong spacing
    }

    // Process each line
    let sceneNumber = 0

    for (let i = 0; i < store.activeProject.lines.length; i++) {
      const line = store.activeProject.lines[i]

      // Calculate line spacing
      const spacing = getLineSpacing(line.type)
      const fontSize = 12
      const lineHeight = fontSize * 1.2
      const threeLinesHeight = lineHeight * 3 // Space for 3 additional lines

      // Check if we need a new page (allow 3 more lines before breaking)
      if (yPosition + spacing.before + lineHeight + spacing.after > pageHeight - bottomMargin - threeLinesHeight) {
        pdf.addPage()
        pageNumber++
        yPosition = topMargin
        pdf.setCharSpace(0) // Reset so first line on new page doesn't get wrong spacing

        // Add page number to footer (except first page)
        if (pageNumber > 1) {
          pdf.setFontSize(10)
          pdf.text(`${pageNumber}.`, pageWidth - rightMargin, pageHeight - bottomMargin + 20, {
            align: 'right',
            charSpace: 0,
          })
        }
      }

      // Add spacing before
      yPosition += spacing.before

      // Skip notes if printNotes is disabled
      if (line.type === 'note' && !printNotes.value) {
        continue // Skip this line entirely
      }

      // Increment scene number for headings
      if (line.type === 'scene-heading') {
        sceneNumber++
      }

      // Render the line
      const rendered = renderLine(pdf, line, yPosition, leftMargin, rightMargin, usableWidth, sceneNumber)
      
      // If renderLine returns null (e.g., note when printNotes is false), skip spacing
      if (rendered === null) {
        continue
      }

      // Move to next line
      yPosition += lineHeight + spacing.after
    }

    // Save the PDF
    pdf.save(filename.value || 'script.pdf')

    visible.value = false
  } catch (error) {
    console.error('PDF Export Error:', error)
    alert('Failed to export PDF. Please try again.')
  } finally {
    isExporting.value = false
  }
}

// Check if text contains Greek characters
const containsGreek = (text) => {
  if (!text) return false
  return /[\u0370-\u03FF\u1F00-\u1FFF]/.test(text)
}

// Script body: left only, no justify, no letter/word spacing. Title page & transitions use center/right.
const SCRIPT_TEXT_OPTIONS = { align: 'left', charSpace: 0 }

/**
 * Render a single line of text to canvas and add as image to PDF.
 * Used for ALL screenplay body text (sluglines, action, dialogue, etc.) so we control
 * alignment (left, never justify), letter-spacing (0), and natural Courier width (no stretch).
 * Canvas is sized to measured text width only - renderer never stretches to fill container.
 */
function renderTextViaCanvas(pdf, text, x, y, options = {}) {
  const fontSize = pdf.getFontSize() || 12
  const font = pdf.internal.getFont()
  const fontStyle = font.fontStyle || 'normal'
  const fontWeight = fontStyle.includes('bold') ? 'bold' : 'normal'
  const fontStyleCSS = fontStyle.includes('italic') ? 'italic' : 'normal'

  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')

  // Standard Courier: measure natural width only (no stretch to container)
  ctx.font = `${fontWeight} ${fontStyleCSS} ${fontSize}pt 'Courier New', Courier, monospace`
  const metrics = ctx.measureText(text)
  const textWidth = Math.ceil(metrics.width)
  const textHeight = fontSize * 1.2

  const scale = 6
  const padding = 4
  canvas.width = Math.round((textWidth + padding * 2) * scale)
  canvas.height = Math.round((textHeight + padding * 2) * scale)
  ctx.scale(scale, scale)

  ctx.fillStyle = '#FFFFFF'
  ctx.fillRect(0, 0, canvas.width / scale, canvas.height / scale)
  ctx.font = `${fontWeight} ${fontStyleCSS} ${fontSize}pt 'Courier New', Courier, monospace`
  ctx.fillStyle = '#000000'
  ctx.textBaseline = 'top'
  ctx.imageSmoothingEnabled = true
  ctx.imageSmoothingQuality = 'high'

  // Left align only for script body; center/right only for title page & transitions
  let textX = padding
  if (options.align === 'right') {
    textX = (canvas.width / scale) - padding
    ctx.textAlign = 'right'
  } else if (options.align === 'center') {
    textX = (canvas.width / scale) / 2
    ctx.textAlign = 'center'
  } else {
    ctx.textAlign = 'left'
  }
  // No letter-spacing: use default. Canvas has no text-align-last; we draw one line so N/A.
  ctx.fillText(text, textX, padding)

  const imgData = canvas.toDataURL('image/png')
  const pdfY = y - fontSize * 0.2
  const imgWidth = Math.round((textWidth + padding * 2) * 0.75 * 100) / 100
  const imgHeight = Math.round((textHeight + padding * 2) * 0.75 * 100) / 100

  let pdfX = x
  if (options.align === 'right') pdfX = x - imgWidth
  else if (options.align === 'center') pdfX = x - imgWidth / 2

  pdf.addImage(imgData, 'PNG', pdfX, pdfY, imgWidth, imgHeight)
}

/**
 * Render text: use canvas for all screenplay body (left-aligned) to avoid jsPDF
 * justify/word-spacing on first/last lines. Use pdf.text() only for title page & transitions (center/right).
 */
const renderText = (pdf, text, x, y, options = {}) => {
  const useCanvas =
    options.align !== 'center' && options.align !== 'right'
  if (useCanvas) {
    try {
      renderTextViaCanvas(pdf, text, x, y, { ...options, align: 'left' })
    } catch (error) {
      console.warn('Canvas render failed, falling back to pdf.text:', error)
      pdf.setCharSpace(0)
      pdf.text(text, x, y, { align: 'left', charSpace: 0 })
    }
    return
  }
  // Title page & transitions: center/right via jsPDF
  pdf.setCharSpace(0)
  pdf.text(text, x, y, {
    align: options.align,
    charSpace: 0,
  })
}

const addTitlePage = (pdf, pageWidth, pageHeight) => {
  const titlePage = store.activeProject.titlePage
  const isTVShow = store.activeProject.format === 'TV Show'

  // TV Show Format
  if (isTVShow) {
    pdf.setFont('courier', 'bold')
    pdf.setFontSize(16)

    // Series Title (centered, top center/dead center)
    const seriesTitle = titlePage.title?.trim() || (language.value === 'el' ? 'ΑΝΩΝΥΜΟ' : 'UNTITLED')
    const seriesTitleLines = pdf.splitTextToSize(seriesTitle.toUpperCase(), 400)
    const centerY = pageHeight / 2
    const seriesTitleY = centerY - 60 // Start above center
    seriesTitleLines.forEach((line, i) => {
      renderText(pdf, line, pageWidth / 2, seriesTitleY + i * 20, { align: 'center', width: 400 })
    })

    // Calculate position after series title
    const seriesTitleBottomY = seriesTitleY + (seriesTitleLines.length * 20)
    
    // Episode Number (1 blank line after series title)
    pdf.setFont('courier', 'normal')
    pdf.setFontSize(12)
    const season = store.activeProject.seasons[store.activeProject.activeSeasonIndex]
    let episodeNumber = ''
    if (season) {
      const episode = season.episodes.find((e) => e.id === store.activeProject.activeEpisodeId)
      if (episode) {
        const episodeIndex = season.episodes.findIndex((e) => e.id === episode.id)
        const seasonNum = String(season.seasonNumber).padStart(2, '0')
        const episodeNum = String(episodeIndex + 1).padStart(2, '0')
        episodeNumber = `Episode S${seasonNum}E${episodeNum}`
      }
    }
    const episodeNumberY = seriesTitleBottomY + 20 // 1 blank line (20pt)
    renderText(pdf, episodeNumber, pageWidth / 2, episodeNumberY, { align: 'center', width: 400 })

    // Episode Title (1 blank line after episode number, in quotes)
    pdf.setFont('courier', 'normal')
    pdf.setFontSize(14)
    const episodeTitlePage = season?.episodes.find((e) => e.id === store.activeProject.activeEpisodeId)?.titlePage
    const episodeTitle = episodeTitlePage?.episodeTitle?.trim() || ''
    const episodeTitleY = episodeNumberY + 20 // 1 blank line (20pt)
    if (episodeTitle) {
      renderText(pdf, `"${episodeTitle.toUpperCase()}"`, pageWidth / 2, episodeTitleY, { align: 'center', width: 400 })
    }

    // Author (centered, below episode title)
    pdf.setFont('courier', 'normal')
    pdf.setFontSize(12)
    const authorY = episodeTitleY + 40 // Space after episode title
    const creditText = titlePage.credit?.trim() || (language.value === 'el' ? 'από' : 'by')
    renderText(pdf, creditText, pageWidth / 2, authorY, { align: 'center', width: 400 })
    const authorText = titlePage.author?.trim() || (language.value === 'el' ? 'ΟΝΟΜΑ ΣΥΓΓΡΑΦΕΑ' : 'AUTHOR NAME')
    renderText(pdf, authorText, pageWidth / 2, authorY + 20, { align: 'center', width: 400 })

    // Draft info (bottom left)
    pdf.setFontSize(11)
    const draftText = titlePage.draft?.trim() || (language.value === 'el' ? 'ΠΡΟΧΕΙΡΟ' : 'DRAFT')
    renderText(pdf, draftText, 108, pageHeight - 100, { width: 200 })

    // Contact info (bottom right)
    pdf.setFontSize(11)
    const contactText = titlePage.contact?.trim() || (language.value === 'el' ? 'ΣΤΟΙΧΕΙΑ ΕΠΙΚΟΙΝΩΝΙΑΣ' : 'CONTACT INFORMATION')
    const contactLines = contactText.split('\n')
    contactLines.forEach((line, i) => {
      renderText(pdf, line, pageWidth - 72, pageHeight - 100 + i * 14, { align: 'right', width: 200 })
    })
  } else {
    // Film/Book Format (Original)
    pdf.setFont('courier', 'bold')
    pdf.setFontSize(14)

    // Title (centered, middle of page) - show placeholder if empty
    const titleText = titlePage.title?.trim() || (language.value === 'el' ? 'ΑΝΩΝΥΜΟ' : 'UNTITLED')
    const titleLines = pdf.splitTextToSize(titleText.toUpperCase(), 400)
    const titleY = pageHeight / 2 - 40
    titleLines.forEach((line, i) => {
      renderText(pdf, line, pageWidth / 2, titleY + i * 20, { align: 'center', width: 400 })
    })

    // Author (centered, below title) - show placeholder if empty
    pdf.setFont('courier', 'normal')
    pdf.setFontSize(12)

    // Calculate the bottom of the title to position author closer
    const titleBottomY = titleY + (titleLines.length * 20)
    const authorY = titleBottomY + 15 // Reduced from 40 to 15 to move closer
    // Use credit field if provided, otherwise use default "by" or "από"
    const creditText = titlePage.credit?.trim() || (language.value === 'el' ? 'από' : 'by')
    renderText(pdf, creditText, pageWidth / 2, authorY, { align: 'center', width: 400 })
    const authorText = titlePage.author?.trim() || (language.value === 'el' ? 'ΟΝΟΜΑ ΣΥΓΓΡΑΦΕΑ' : 'AUTHOR NAME')
    renderText(pdf, authorText, pageWidth / 2, authorY + 20, { align: 'center', width: 400 })

    // Draft info (bottom left) - show placeholder if empty
    pdf.setFontSize(11)
    const draftText = titlePage.draft?.trim() || (language.value === 'el' ? 'ΠΡΟΧΕΙΡΟ' : 'DRAFT')
    renderText(pdf, draftText, 108, pageHeight - 100, { width: 200 }) // Moved down from 144 to 100 (closer to bottom)

    // Contact info (bottom right) - show placeholder if empty
    pdf.setFontSize(11)
    const contactText = titlePage.contact?.trim() || (language.value === 'el' ? 'ΣΤΟΙΧΕΙΑ ΕΠΙΚΟΙΝΩΝΙΑΣ' : 'CONTACT INFORMATION')
    const contactLines = contactText.split('\n')
    contactLines.forEach((line, i) => {
      renderText(pdf, line, pageWidth - 72, pageHeight - 100 + i * 14, { align: 'right', width: 200 }) // Moved down from 144 to 100 (closer to bottom)
    })
  }
}

const renderLine = (pdf, line, y, leftMargin, rightMargin, usableWidth, sceneNumber) => {
  const content = line.content || ''

  // Keep character spacing at 0 so first/last line of block or page don't get big gaps
  pdf.setCharSpace(0)

  // Set formatting based on line type
  switch (line.type) {
    case 'scene-heading':
      pdf.setFont('courier', 'bold')
      pdf.setFontSize(12)

      // Add scene number if enabled
      if (includeSceneNumbers.value) {
        renderText(pdf, `${sceneNumber}.`, leftMargin - 40, y, { width: 40 })
      }

      const headingText = language.value === 'el' ? content.toUpperCase() : content.toUpperCase()
      const headingLines = pdf.splitTextToSize(headingText, usableWidth)
      headingLines.forEach((textLine, i) => {
        renderText(pdf, textLine, leftMargin, y + i * 14, { width: usableWidth })
      })
      break

    case 'action':
      pdf.setFont('courier', 'normal')
      pdf.setFontSize(12)
      const actionLines = pdf.splitTextToSize(content, usableWidth)
      actionLines.forEach((textLine, i) => {
        renderText(pdf, textLine, leftMargin, y + i * 14, { width: usableWidth })
      })
      break

    case 'character':
      pdf.setFont('courier', 'normal')
      pdf.setFontSize(12)
      const charX = leftMargin + 170 // Character indent (reduced from 240 to move left)
      const charText = language.value === 'el' ? content.toUpperCase() : content.toUpperCase()
      renderText(pdf, charText, charX, y, { width: usableWidth - 170 })
      break

    case 'dialogue':
      pdf.setFont('courier', 'normal')
      pdf.setFontSize(12)
      const dialogueX = leftMargin + 120 // Dialogue indent
      const dialogueWidth = usableWidth - 240
      const dialogueLines = pdf.splitTextToSize(content, dialogueWidth)
      dialogueLines.forEach((textLine, i) => {
        renderText(pdf, textLine, dialogueX, y + i * 14, { width: dialogueWidth })
      })
      break

    case 'parenthetical':
      pdf.setFont('courier', 'italic')
      pdf.setFontSize(12)
      const parenX = leftMargin + 160
      renderText(pdf, content, parenX, y, { width: usableWidth - 160 })
      break

    case 'transition':
      pdf.setFont('courier', 'bold')
      pdf.setFontSize(12)
      const pageWidth = 612
      const transitionText = language.value === 'el' ? content.toUpperCase() : content.toUpperCase()
      renderText(pdf, transitionText, pageWidth - rightMargin, y, { align: 'right', width: 200 })
      break

    case 'note':
      // Only render notes if printNotes is enabled
      if (printNotes.value) {
        pdf.setFont('courier', 'italic')
        pdf.setFontSize(12)
        // Remove "//" prefix for display
        const noteContent = content.startsWith('//') ? content.substring(2).trim() : content
        const noteLines = pdf.splitTextToSize(noteContent, usableWidth)
        noteLines.forEach((textLine, i) => {
          renderText(pdf, textLine, leftMargin, y + i * 14, { width: usableWidth })
        })
        return true // Indicate line was rendered
      }
      // If printNotes is false, skip rendering
      return null

    default:
      pdf.setFont('courier', 'normal')
      pdf.setFontSize(12)
      renderText(pdf, content, leftMargin, y, { width: usableWidth })
  }
}

const getLineSpacing = (type) => {
  switch (type) {
    case 'scene-heading':
      return { before: 24, after: 12 }
    case 'character':
      return { before: 12, after: 0 }
    case 'dialogue':
      return { before: 0, after: 12 }
    case 'note':
      return { before: 0, after: 12 }
    case 'action':
      return { before: 0, after: 12 }
    default:
      return { before: 0, after: 0 }
  }
}
</script>

<style scoped>
.export-options {
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding: 10px 0;
}

.option-field {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.option-field label {
  font-weight: 600;
  font-size: 14px;
  color: #333;
}

:global(body.dark-mode) .option-field label {
  color: #e0e0e0;
}

.checkbox-wrapper {
  display: flex;
  align-items: center;
  gap: 10px;
}

.checkbox-wrapper input[type='checkbox'] {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.checkbox-wrapper label {
  font-weight: 400;
  cursor: pointer;
  user-select: none;
}

.option-explanation {
  font-size: 12px;
  color: #666;
  font-style: italic;
  margin-top: 4px;
  padding-left: 28px;
}

:global(body.dark-mode) .option-explanation {
  color: #999;
}

.w-full {
  width: 100%;
}

.language-select {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  background: white;
  cursor: pointer;
  transition: border-color 0.2s ease;
}

.language-select:hover {
  border-color: #999;
}

.language-select:focus {
  outline: none;
  border-color: #666;
  box-shadow: 0 0 0 2px rgba(0, 0, 0, 0.1);
}

:global(body.dark-mode) .language-select {
  background: #2a2a2a;
  border-color: #444;
  color: #e0e0e0;
}

:global(body.dark-mode) .language-select:hover {
  border-color: #666;
}

.export-progress {
  display: flex;
  align-items: center;
  gap: 15px;
  padding: 20px;
  background: #f5f5f5;
  border-radius: 8px;
  margin-top: 10px;
}

:global(body.dark-mode) .export-progress {
  background: #2a2a2a;
}

.progress-spinner {
  width: 30px;
  height: 30px;
  border: 3px solid #e0e0e0;
  border-top-color: #666;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.export-progress p {
  margin: 0;
  color: #666;
  font-size: 14px;
}

:global(body.dark-mode) .export-progress p {
  color: #aaa;
}
</style>
