<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'
import * as htmlToImage from 'html-to-image'

// STATE
const rawPrompt = ref('')
const generatedPrompt = ref('')
const showCard = computed(() => generatedPrompt.value.length > 0)
const maxChars = 1200
const layout = ref('square') // square | vertical
const pages = ref([])
const exporting = ref(false)

// GRADIENTS
const gradients = [
  'linear-gradient(135deg,#6366f1,#8b5cf6,#ec4899)',
  'linear-gradient(135deg,#06b6d4,#3b82f6,#6366f1)',
  'linear-gradient(135deg,#f59e0b,#ef4444,#ec4899)',
  'linear-gradient(135deg,#10b981,#06b6d4,#3b82f6)',
  'linear-gradient(135deg,#a855f7,#6366f1,#3b82f6)'
]
const gradientIndex = ref(0)

// DERIVED
const charCount = computed(() => rawPrompt.value.length)
const charPercent = computed(() => Math.min(100, (charCount.value / maxChars) * 100))
const layoutLabel = computed(() => layout.value === 'square' ? '小红书 / Instagram' : '抖音 / TikTok')
const shareUrl = computed(() => {
  if (!generatedPrompt.value) return location.origin + location.pathname
  const encoded = base64UrlEncode(generatedPrompt.value)
  return `${location.origin}${location.pathname}?p=${encoded}`
})

// COPY FLAGS
const copiedPrompt = ref(false)
const copiedUrl = ref(false)

// FUNCTIONS
function cycleGradient() { gradientIndex.value = (gradientIndex.value + 1) % gradients.length }
function generateCard() {
  if (!rawPrompt.value.trim()) return
  generatedPrompt.value = rawPrompt.value.trim()
  buildPages()
  cycleGradient()
  updateUrlParam(generatedPrompt.value)
  requestAnimationFrame(() => document.querySelector('.export-page')?.scrollIntoView({ behavior: 'smooth' }))
}
async function copyPrompt() {
  if (!generatedPrompt.value) return
  await navigator.clipboard.writeText(generatedPrompt.value)
  copiedPrompt.value = true; setTimeout(()=>copiedPrompt.value=false,1500)
}
async function copyUrl() {
  await navigator.clipboard.writeText(shareUrl.value)
  copiedUrl.value = true; setTimeout(()=>copiedUrl.value=false,1500)
}
function updateUrlParam(p) {
  const encoded = base64UrlEncode(p)
  history.replaceState(null, '', `${location.pathname}?p=${encoded}`)
}
function base64UrlEncode(str){return btoa(unescape(encodeURIComponent(str))).replace(/\+/g,'-').replace(/\//g,'_').replace(/=+$/,'')}
function base64UrlDecode(str){try{str=str.replace(/-/g,'+').replace(/_/g,'/');while(str.length%4)str+='=';return decodeURIComponent(escape(atob(str)))}catch{return ''}}
function hashString(str){let h=0;for(let i=0;i<str.length;i++)h=(Math.imul(31,h)+str.charCodeAt(i))|0;return h}
function loadFromUrl(){const p=new URLSearchParams(location.search).get('p');if(p){const d=base64UrlDecode(p);if(d){rawPrompt.value=d;generatedPrompt.value=d;gradientIndex.value=Math.abs(hashString(d))%gradients.length;buildPages()}}}
function buildPages(){const text=generatedPrompt.value;if(!text){pages.value=[];return}const lines=text.split(/\n+/);const limit=layout.value==='square'?420:650;const out=[];let buf='';for(const ln of lines){const add=(buf?'\n':'')+ln;if((buf+add).length>limit){if(buf)out.push(buf);buf=ln;while(buf.length>limit){out.push(buf.slice(0,limit));buf=buf.slice(limit)}}else{buf+=add}}if(buf)out.push(buf);pages.value=out}
function cardStyle(i){return { background: gradients[(gradientIndex.value + i) % gradients.length] }}
async function exportImages(){ if(!showCard.value) return; exporting.value=true; await nextTick(); try { const nodes=[...document.querySelectorAll('.export-page')]; let idx=1; for(const n of nodes){ const data=await htmlToImage.toPng(n,{pixelRatio:2}); const a=document.createElement('a'); a.download=`prompt-card-${layout.value}-${idx}.png`; a.href=data; a.click(); idx++; } } catch(e){ console.error(e); } finally { exporting.value=false } }

// WATCHERS / LIFECYCLE
onMounted(loadFromUrl)
watch(rawPrompt,v=>{ if(v.length>maxChars) rawPrompt.value=v.slice(0,maxChars) })
watch(generatedPrompt, buildPages)
watch(layout, buildPages)
</script>

<template>
  <div class="w-full min-h-screen bg-gradient-to-br from-slate-900 via-indigo-950 to-fuchsia-950/60 text-slate-100">
    <div class="mx-auto max-w-6xl px-5 py-10 space-y-12">
      <header class="text-center space-y-3">
        <h1 class="text-3xl md:text-4xl font-bold tracking-tight bg-gradient-to-r from-indigo-400 via-fuchsia-400 to-rose-400 bg-clip-text text-transparent">Prompt Sharing Card</h1>
        <p class="text-sm md:text-base text-slate-400">生成适配 <span class="text-indigo-300 font-medium">小红书 / Instagram</span> 与 <span class="text-fuchsia-300 font-medium">抖音 / TikTok</span> 的多页分享图</p>
      </header>
      <section class="relative rounded-2xl bg-slate-900/70 ring-1 ring-white/10 backdrop-blur-md shadow-xl overflow-hidden">
        <div class="absolute inset-0 pointer-events-none bg-[radial-gradient(circle_at_15%_20%,rgba(255,255,255,0.07),transparent_60%)]"></div>
        <div class="p-5 md:p-7 space-y-4 relative">
          <div class="flex flex-col md:flex-row gap-5">
            <div class="flex-1">
              <textarea v-model="rawPrompt" :placeholder="'Describe the image / 输入你的提示词...'" @keydown.meta.enter.prevent="generateCard" @keydown.ctrl.enter.prevent="generateCard" class="w-full h-56 resize-y rounded-xl bg-slate-800/70 ring-1 ring-white/10 focus:ring-2 focus:ring-indigo-500/60 focus:outline-none px-4 py-3 text-sm leading-relaxed placeholder:text-slate-500 font-medium text-slate-200 shadow-inner"></textarea>
              <div class="flex items-center justify-between mt-2 text-[11px] uppercase tracking-wider text-slate-500">
                <span>⌘/Ctrl + Enter 生成</span>
                <span :class="['font-semibold', charPercent>85 ? (charPercent>=100 ? 'text-rose-400':'text-amber-400'):'text-slate-400']">{{ charCount }} / {{ maxChars }}</span>
              </div>
            </div>
            <div class="w-full md:w-64 flex flex-col gap-3">
              <div class="grid grid-cols-2 rounded-xl bg-slate-800/60 p-1 ring-1 ring-white/10 text-xs font-medium overflow-hidden">
                <button type="button" @click="layout='square'" :class="['py-1.5 rounded-lg transition', layout==='square' ? 'bg-indigo-500/80 text-white shadow':'text-slate-400 hover:text-slate-200']">方形 1:1</button>
                <button type="button" @click="layout='vertical'" :class="['py-1.5 rounded-lg transition', layout==='vertical' ? 'bg-indigo-500/80 text-white shadow':'text-slate-400 hover:text-slate-200']">竖屏 9:16</button>
              </div>
              <button @click="generateCard" :disabled="!rawPrompt.trim()" class="rounded-xl bg-gradient-to-r from-indigo-500 via-fuchsia-500 to-rose-500 px-4 py-2.5 text-sm font-semibold text-white shadow hover:brightness-110 disabled:opacity-40 transition">{{ showCard ? '更新卡片' : '生成卡片' }}</button>
              <div class="flex gap-2 flex-wrap" v-if="showCard">
                <button @click="copyUrl" class="flex-1 text-xs rounded-lg px-3 py-2 bg-slate-800/70 ring-1 ring-white/10 hover:ring-indigo-400/40 transition">{{ copiedUrl ? '已复制链接' : '复制链接' }}</button>
                <button @click="exportImages" :disabled="exporting" class="flex-1 text-xs rounded-lg px-3 py-2 bg-slate-800/70 ring-1 ring-white/10 hover:ring-indigo-400/40 disabled:opacity-40 transition">{{ exporting ? '导出中...' : '导出图片' }}</button>
                <button @click="copyPrompt" class="flex-1 text-xs rounded-lg px-3 py-2 bg-slate-800/70 ring-1 ring-white/10 hover:ring-indigo-400/40 transition">{{ copiedPrompt ? '已复制文本' : '复制文本' }}</button>
              </div>
            </div>
          </div>
        </div>
      </section>
      <transition name="fade-slide">
        <section v-if="showCard" class="space-y-10">
          <div class="grid gap-10" :class="layout==='vertical' ? 'md:grid-cols-2 lg:grid-cols-3' : 'md:grid-cols-3 xl:grid-cols-4'">
            <article v-for="(p,i) in pages" :key="i" class="relative group rounded-3xl shadow-2xl overflow-hidden export-page border border-white/10" :style="cardStyle(i)">
              <div class="absolute inset-0 opacity-[0.15] mix-blend-soft-light bg-[radial-gradient(circle_at_30%_20%,rgba(255,255,255,0.9),transparent_60%)]"></div>
              <div class="flex flex-col h-full p-6 md:p-7 backdrop-blur-[1px]">
                <div class="flex items-center justify-between text-[10px] tracking-widest font-semibold uppercase text-white/80">
                  <span>Prompt {{ i+1 }} / {{ pages.length }}</span>
                  <span class="px-2 py-1 rounded-full bg-white/20 text-[10px]">{{ layoutLabel }}</span>
                </div>
                <div class="mt-5 whitespace-pre-wrap font-medium leading-relaxed text-sm tracking-wide text-white/90 flex-1 overflow-hidden" style="word-break: break-word;">{{ p }}</div>
                <div class="mt-6 flex gap-2">
                  <button @click="copyPrompt" class="flex-1 text-xs rounded-lg px-3 py-2 bg-white/15 hover:bg-white/25 text-white font-medium transition">复制文本</button>
                  <button @click="copyUrl" class="flex-1 text-xs rounded-lg px-3 py-2 bg-white/15 hover:bg-white/25 text-white font-medium transition">分享链接</button>
                </div>
              </div>
            </article>
          </div>
        </section>
      </transition>
      <footer class="pt-4 text-center text-[11px] tracking-wide text-slate-500 space-y-1">
        <p>多页切分 · 2x 高清导出 · URL 分享</p>
      </footer>
    </div>
  </div>
</template>

<style scoped>
.fade-slide-enter-active,.fade-slide-leave-active{transition:all .55s cubic-bezier(.16,.8,.27,1)}
.fade-slide-enter-from,.fade-slide-leave-to{opacity:0;transform:translateY(20px) scale(.98)}
</style>
