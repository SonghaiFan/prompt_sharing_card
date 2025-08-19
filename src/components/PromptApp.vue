<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'
import * as htmlToImage from 'html-to-image'

// Core state
const rawPrompt = ref('')
const generatedPrompt = ref('')
const showCard = computed(() => generatedPrompt.value.length > 0)
const maxChars = 1200

// Layout & export
const layout = ref('square') // 'square' | 'vertical'
const exporting = ref(false)
const pages = ref([]) // chunked prompt lines
const pageHeightPx = 1080 // for vertical reference (not strictly needed)

const layoutLabel = computed(() => layout.value === 'square' ? '小红书 / Instagram 方形' : '抖音 / TikTok 竖屏')

// Gradient palette
const gradients = [
  'linear-gradient(135deg,#6366f1,#8b5cf6,#ec4899)',
  'linear-gradient(135deg,#06b6d4,#3b82f6,#6366f1)',
  'linear-gradient(135deg,#f59e0b,#ef4444,#ec4899)',
  'linear-gradient(135deg,#10b981,#06b6d4,#3b82f6)',
  'linear-gradient(135deg,#a855f7,#6366f1,#3b82f6)'
]
const gradientIndex = ref(0)

// Copy feedback flags
const copiedPrompt = ref(false)
const copiedUrl = ref(false)

// Character stats
const charCount = computed(() => rawPrompt.value.length)
const charPercent = computed(() => Math.min(100, (charCount.value / maxChars) * 100))

// Share URL derived from encoded prompt
const shareUrl = computed(() => {
  if (!generatedPrompt.value) return window.location.origin + window.location.pathname
  const encoded = base64UrlEncode(generatedPrompt.value)
  return `${window.location.origin}${window.location.pathname}?p=${encoded}`
})

function cycleGradient() {
  gradientIndex.value = (gradientIndex.value + 1) % gradients.length
}
function generateCard() {
  if (!rawPrompt.value.trim()) return
  generatedPrompt.value = rawPrompt.value.trim()
  buildPages()
  cycleGradient()
  updateUrlParam(generatedPrompt.value)
  requestAnimationFrame(() => {
    document.querySelector('.prompt-card')?.scrollIntoView({ behavior: 'smooth', block: 'start' })
  })
}
async function copyPrompt() {
  if (!generatedPrompt.value) return
  await navigator.clipboard.writeText(generatedPrompt.value)
  copiedPrompt.value = true
  setTimeout(() => (copiedPrompt.value = false), 1500)
}
async function copyUrl() {
  await navigator.clipboard.writeText(shareUrl.value)
  copiedUrl.value = true
  setTimeout(() => (copiedUrl.value = false), 1500)
}
function updateUrlParam(prompt) {
  const encoded = base64UrlEncode(prompt)
  const newUrl = `${window.location.pathname}?p=${encoded}`
  window.history.replaceState(null, '', newUrl)
}
function base64UrlEncode(str) {
  return btoa(unescape(encodeURIComponent(str)))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '')
}
function base64UrlDecode(str) {
  try {
    str = str.replace(/-/g, '+').replace(/_/g, '/')
    while (str.length % 4) str += '='
    return decodeURIComponent(escape(atob(str)))
  } catch {
    return ''
  }
}
function loadFromUrl() {
  const params = new URLSearchParams(window.location.search)
  const p = params.get('p')
  if (p) {
    const decoded = base64UrlDecode(p)
    if (decoded) {
      rawPrompt.value = decoded
      generatedPrompt.value = decoded
      gradientIndex.value = Math.abs(hashString(decoded)) % gradients.length
    }
  }
}
function hashString(str) {
  let h = 0
  for (let i = 0; i < str.length; i++) h = (Math.imul(31, h) + str.charCodeAt(i)) | 0
  return h
}
onMounted(loadFromUrl)
watch(rawPrompt, v => { if (v.length > maxChars) rawPrompt.value = v.slice(0, maxChars) })

watch(generatedPrompt, () => buildPages())

function buildPages() {
  // Split by line, approximate characters per page based on layout.
  const text = generatedPrompt.value
  if (!text) { pages.value = []; return }
  const lines = text.split(/\n+/)
  const maxCharsPerPage = layout.value === 'square' ? 420 : 650
  const chunked = []
  let buf = ''
  for (const line of lines) {
    const add = (buf ? '\n' : '') + line
    if ((buf + add).length > maxCharsPerPage) {
      if (buf) chunked.push(buf)
      buf = line
      if (buf.length > maxCharsPerPage) {
        // hard split long single line
        while (buf.length > maxCharsPerPage) {
          chunked.push(buf.slice(0, maxCharsPerPage))
          buf = buf.slice(maxCharsPerPage)
        }
      }
    } else {
      buf += add
    }
  }
  if (buf) chunked.push(buf)
  pages.value = chunked
}

async function exportImages() {
  if (!showCard.value) return
  exporting.value = true
  await nextTick()
  try {
    const nodes = Array.from(document.querySelectorAll('.export-page'))
    let idx = 1
    for (const node of nodes) {
      // scale up for quality
      const dataUrl = await htmlToImage.toPng(node, { pixelRatio: 2 })
      const link = document.createElement('a')
      link.download = `prompt-card-${layout.value}-${idx}.png`
      link.href = dataUrl
      link.click()
      idx++
    }
  } catch (e) {
    console.error('Export failed', e)
  } finally {
    exporting.value = false
  }
}
</script>

<template>
  <div class="layout">
    <header class="app-header">
      <h1 class="gradient-text">Prompt Sharing Card</h1>
      <p class="tagline">Create & share beautiful text-to-image prompt cards instantly.</p>
    </header>
    <section class="input-panel" :class="{ elevated: !showCard }">
      <label for="prompt" class="sr-only">Prompt</label>
      <textarea id="prompt" v-model="rawPrompt" :placeholder="'Describe the image you imagine...'" @keydown.meta.enter.prevent="generateCard" @keydown.ctrl.enter.prevent="generateCard"></textarea>
      <div class="input-meta">
        <div class="char-bar" :style="{ '--pct': charPercent + '%' }"></div>
        <span class="count" :class="{ near: charPercent > 85, full: charPercent >= 100 }">{{ charCount }} / {{ maxChars }}</span>
      </div>
      <div class="actions">
        <button class="primary" :disabled="!rawPrompt.trim()" @click="generateCard">Generate Card</button>
        <button class="ghost" v-if="showCard" @click="copyUrl">{{ copiedUrl ? 'URL Copied!' : 'Copy Share URL' }}</button>
        <select v-if="showCard" v-model="layout" class="layout-select" @change="buildPages">
          <option value="square">方形 (1:1)</option>
          <option value="vertical">竖屏 (9:16)</option>
        </select>
        <button class="ghost" v-if="showCard" :disabled="exporting" @click="exportImages">{{ exporting ? '导出中...' : '导出图片' }}</button>
      </div>
      <p class="hint">Press ⌘⏎ / Ctrl⏎ to generate.</p>
    </section>
    <transition name="fade-slide">
      <section v-if="showCard" class="card-wrapper">
        <div class="pages" :class="layout">
          <article v-for="(p, i) in pages" :key="i" class="prompt-card export-page" :style="{ background: gradients[(gradientIndex + i) % gradients.length] }">
            <div class="card-inner" :class="layout">
              <header class="card-head">
                <h2 class="card-title">Prompt ({{ i + 1 }}/{{ pages.length }})</h2>
                <span class="layout-badge">{{ layoutLabel }}</span>
              </header>
              <p class="prompt-text" v-text="p" />
              <div class="share-row">
                <button class="copy-btn" @click="copyPrompt">{{ copiedPrompt ? 'Copied!' : 'Copy Text' }}</button>
                <button class="copy-btn" @click="copyUrl">{{ copiedUrl ? 'URL Copied!' : 'Share URL' }}</button>
              </div>
            </div>
          </article>
        </div>
      </section>
    </transition>
    <section class="how-it-works">
      <h3>How it works</h3>
      <ol>
        <li>Enter a text-to-image prompt</li>
        <li>Generate a beautiful gradient card</li>
        <li>Share the unique URL</li>
        <li>Others view & copy your prompt</li>
        <li>Copy buttons give instant feedback</li>
      </ol>
    </section>
  </div>
</template>

<style scoped>
.layout {
  max-width: 960px;
  margin: 0 auto;
  padding: clamp(1rem, 2.5vw, 2.5rem);
  display: flex;
  flex-direction: column;
  gap: 2.5rem;
}
.app-header {
  text-align: center;
}
.gradient-text {
  font-size: clamp(1.9rem, 4.5vw, 3.25rem);
  background: linear-gradient(90deg, #6366f1, #8b5cf6, #ec4899, #ef4444);
  background-clip: text;
  -webkit-background-clip: text;
  color: transparent;
  font-weight: 700;
  letter-spacing: -0.5px;
  animation: hue 12s linear infinite;
  background-size: 300% 100%;
}
.tagline {
  margin-top: 0.5rem;
  color: var(--c-subtle);
  font-size: 0.95rem;
}
:root {
  --c-bg: #0f1115;
  --c-panel: #1b1f27;
  --c-border: #2a303b;
  --c-text: #e6eaf1;
  --c-subtle: #94a3b8;
  --c-accent: #6366f1;
  --radius: 18px;
}
@media (prefers-color-scheme: light) {
  :root {
    --c-bg: #f5f7fa;
    --c-panel: #ffffff;
    --c-border: #e2e8f0;
    --c-text: #1e293b;
    --c-subtle: #64748b;
  }
}
body {
  background: radial-gradient(circle at 20% 20%, #1e2530, #0f1115 60%) fixed;
}
.input-panel {
  background: var(--c-panel);
  border: 1px solid var(--c-border);
  padding: 1.25rem 1.25rem 1.5rem;
  border-radius: var(--radius);
  position: relative;
  box-shadow: 0 4px 24px -4px rgba(0, 0, 0, 0.4), 0 2px 4px rgba(0, 0, 0, 0.3);
  transition: box-shadow 0.4s, transform 0.4s;
}
.input-panel.elevated {
  transform: translateY(4px);
}
.input-panel:focus-within {
  box-shadow: 0 6px 28px -4px rgba(0, 0, 0, 0.5), 0 4px 10px rgba(0, 0, 0, 0.35);
}
textarea {
  width: 100%;
  min-height: 140px;
  resize: vertical;
  background: linear-gradient(145deg, #151a21, #1e2530);
  color: var(--c-text);
  border: 1px solid var(--c-border);
  border-radius: 14px;
  padding: 1rem 1.1rem;
  font: 500 1rem/1.5 system-ui, -apple-system, Segoe UI, Roboto, Helvetica,
    Arial, sans-serif;
  outline: none;
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.03);
  transition: border-color 0.3s, box-shadow 0.3s, background 0.4s;
}
textarea:focus {
  border-color: var(--c-accent);
  box-shadow: 0 0 0 1px var(--c-accent), 0 0 0 4px rgba(99, 102, 241, 0.25);
}
textarea::placeholder {
  color: var(--c-subtle);
  opacity: 0.5;
}
.input-meta {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  margin-top: 0.5rem;
  position: relative;
}
.char-bar {
  position: absolute;
  left: 0;
  top: 0;
  height: 3px;
  width: 100%;
  background: linear-gradient(90deg, var(--c-accent), #ec4899);
  transform: scaleX(calc(var(--pct) / 100));
  transform-origin: left;
  transition: transform 0.3s;
  border-radius: 2px;
  opacity: 0.9;
}
.count {
  font-size: 0.75rem;
  font-weight: 600;
  letter-spacing: 0.5px;
  padding: 0.25rem 0.5rem;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.04);
  color: var(--c-subtle);
  backdrop-filter: blur(4px);
}
.count.near {
  color: #f59e0b;
}
.count.full {
  color: #ef4444;
}
.actions {
  display: flex;
  gap: 0.75rem;
  margin-top: 1rem;
  flex-wrap: wrap;
}
button {
  cursor: pointer;
  font: 600 0.95rem/1 system-ui, -apple-system, Segoe UI, Roboto, sans-serif;
  border-radius: 12px;
  border: 1px solid var(--c-border);
  background: linear-gradient(145deg, #202733, #1b212b);
  color: var(--c-text);
  padding: 0.85rem 1.25rem;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  position: relative;
  overflow: hidden;
  transition: background 0.4s, border-color 0.4s, transform 0.25s;
}
button.primary {
  background: linear-gradient(90deg, #6366f1, #8b5cf6, #ec4899);
  background-size: 200% 100%;
  border: none;
  color: #fff;
  box-shadow: 0 6px 18px -6px rgba(99, 102, 241, 0.6);
}
button.primary:hover:not(:disabled) {
  background-position: 100% 0;
}
button:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}
button.ghost {
  background: rgba(99, 102, 241, 0.12);
  border-color: rgba(99, 102, 241, 0.4);
  color: var(--c-accent);
}
button.ghost:hover {
  background: rgba(99, 102, 241, 0.2);
}
button:active:not(:disabled) {
  transform: translateY(2px);
}
.hint {
  margin: 0.5rem 0 0;
  font-size: 0.7rem;
  text-transform: uppercase;
  letter-spacing: 0.12em;
  color: var(--c-subtle);
  opacity: 0.7;
}
.card-wrapper {
  display: flex;
  justify-content: center;
}
.prompt-card {
  width: min(100%, 680px);
  border-radius: 28px;
  padding: 0;
  position: relative;
  overflow: hidden;
  color: #fff;
  backdrop-filter: blur(6px);
  box-shadow: 0 12px 40px -8px rgba(0, 0, 0, 0.5),
    0 4px 12px -2px rgba(0, 0, 0, 0.4);
  animation: appear 0.7s cubic-bezier(0.16, 0.8, 0.27, 1);
}
.pages.square .prompt-card { aspect-ratio: 1 / 1; display:flex; }
.pages.vertical .prompt-card { aspect-ratio: 9 / 16; display:flex; max-width:420px; }
.card-inner.vertical { padding: 2rem 1.4rem 2.2rem; }
.layout-select { border:1px solid var(--c-border); background:#1e2530; color:var(--c-text); border-radius:10px; padding:0.6rem 0.9rem; font-size:0.8rem; }
.layout-badge { font-size:0.6rem; letter-spacing:0.1em; background:rgba(255,255,255,.25); padding:0.35rem 0.6rem; border-radius:999px; text-transform:uppercase; }
.card-head { display:flex; align-items:center; justify-content:space-between; gap:0.75rem; }
.prompt-card::before {
  content: "";
  position: absolute;
  inset: 0;
  background: radial-gradient(
    circle at 30% 30%,
    rgba(255, 255, 255, 0.25),
    transparent 60%
  );
  mix-blend-mode: overlay;
  pointer-events: none;
}
.card-inner {
  position: relative;
  padding: 2.4rem clamp(1.4rem, 3vw, 2.6rem) 2.8rem;
  display: flex;
  flex-direction: column;
  gap: 1.75rem;
}
.card-title {
  margin: 0;
  font-size: 1.1rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  opacity: 0.9;
  font-weight: 700;
}
.prompt-text {
  white-space: pre-wrap;
  font-size: clamp(1rem, 1.05rem + 0.2vw, 1.15rem);
  line-height: 1.5;
  font-weight: 500;
  letter-spacing: 0.3px;
  text-shadow: 0 2px 6px rgba(0, 0, 0, 0.25);
}
.share-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}
.copy-btn {
  flex: 1 1 160px;
  background: rgba(255, 255, 255, 0.15);
  border: 1px solid rgba(255, 255, 255, 0.35);
  color: #fff;
  font-weight: 600;
  backdrop-filter: blur(4px);
}
.copy-btn:hover {
  background: rgba(255, 255, 255, 0.25);
}
.copy-btn:active {
  transform: translateY(2px);
}
.how-it-works {
  background: var(--c-panel);
  border: 1px solid var(--c-border);
  border-radius: var(--radius);
  padding: 1.5rem 1.75rem 1.9rem;
  line-height: 1.55;
}
.how-it-works h3 {
  margin: 0 0 1rem;
  font-size: 1rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--c-accent);
}
.how-it-works ol {
  margin: 0;
  padding-left: 1.1rem;
  display: grid;
  gap: 0.35rem;
  font-weight: 500;
}
@keyframes hue {
  to {
    filter: hue-rotate(360deg);
  }
}
@keyframes appear {
  0% {
    opacity: 0;
    transform: translateY(24px) scale(0.96);
  }
  70% {
    opacity: 1;
  }
  100% {
    transform: translateY(0) scale(1);
  }
}
.fade-slide-enter-active,
.fade-slide-leave-active {
  transition: all 0.55s cubic-bezier(0.16, 0.8, 0.27, 1);
}
.fade-slide-enter-from,
.fade-slide-leave-to {
  opacity: 0;
  transform: translateY(20px) scale(0.98);
}
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  border: 0;
}
@media (max-width: 640px) {
  .card-inner {
    padding: 1.9rem 1.4rem 2.2rem;
  }
  .layout {
    gap: 2rem;
  }
  .prompt-card {
    border-radius: 24px;
  }
}
</style>
