<template>
  <div class="ra-root" :class="{ 'ra-open': isOpen, 'ra-big': isBig, 'ra-drop': isDragging }"
       @dragenter.prevent="isDragging = true" @dragover.prevent="isDragging = true"
       @dragleave.prevent="isDragging = false" @drop.prevent="onDrop">
    <div v-if="isBig" class="ra-backdrop" @click="setBig(false)"></div>

    <section v-show="isOpen" class="ra-card" aria-label="AI Renovation Assistant">
      <header class="ra-head">
        <h3>AI Renovation Assistant</h3>
        <div class="head-actions">
          <button class="ra-btn" type="button" @click="newChat" :disabled="busy">New chat</button>
          <button class="ra-btn ra-x ra-expand" type="button" @click="setBig(!isBig)"
                  :aria-label="isBig ? 'Make chat smaller' : 'Enlarge chat'">
            <svg viewBox="0 0 24 24" aria-hidden="true"><polyline v-if="!isBig" points="15 3 21 3 21 9"/><polyline v-if="!isBig" points="9 21 3 21 3 15"/><line v-if="!isBig" x1="21" y1="3" x2="14" y2="10"/><line v-if="!isBig" x1="3" y1="21" x2="10" y2="14"/><polyline v-if="isBig" points="4 14 10 14 10 20"/><polyline v-if="isBig" points="20 10 14 10 14 4"/><line v-if="isBig" x1="14" y1="10" x2="21" y2="3"/><line v-if="isBig" x1="3" y1="21" x2="10" y2="14"/></svg>
          </button>
          <button class="ra-btn ra-x" type="button" @click="setOpen(false)" aria-label="Close chat">×</button>
        </div>
      </header>

      <div ref="logEl" class="ra-log" role="log" aria-live="polite">
        <template v-if="messages.length">
          <div v-for="(m, i) in messages" :key="i" class="ra-msg" :class="[m.role, { err: m.err }]">
            <div class="message-text" v-html="formatText(m.text)"></div>
            <div v-if="m.image" class="ra-gen">
              <a :href="m.image.url" target="_blank" rel="noopener"><img :src="m.image.url" alt="AI preview of your room with Widebed products"></a>
              <div class="ra-gen-note">AI-generated preview for illustration only. Colours, size and details may differ from the real products.
                <a :href="m.image.url" :download="`widebed-preview.${m.image.ext || 'png'}`">Download</a>
              </div>
            </div>
            <div v-if="m.packages?.length" class="ra-cards pkg"><ProductCard v-for="(p, j) in m.packages" :key="j" :product="p" /></div>
            <div v-if="m.products?.length" class="ra-cards"><ProductCard v-for="(p, j) in m.products" :key="j" :product="p" /></div>
            <div v-if="m.files?.length" class="ra-files">
              <div v-for="(f, j) in m.files" :key="j" class="ra-file">
                <img v-if="f.type?.startsWith('image/') && f.url" :src="f.url" alt="">
                <div v-else class="ra-pdf">PDF</div><span>{{ f.name }}</span>
              </div>
            </div>
          </div>
        </template>
        <div v-else class="ra-msg bot">{{ greeting }}</div>
        <div v-if="busy" class="ra-typing">
          <span class="ra-dots"><i></i><i></i><i></i></span>{{ typingText }}
          <div v-if="hasPending" class="ra-bar"><b :style="{ width: progress + '%' }"></b></div>
        </div>
      </div>

      <div v-if="suggestions.length && !busy" class="ra-suggest">
        <button v-for="(s, i) in suggestions" :key="i" class="ra-chip" type="button" @click="send(s)">{{ s }}</button>
      </div>
      <div v-if="pending.length" class="ra-pending">
        <div v-for="f in pending" :key="f.id" class="ra-file">
          <img v-if="f.type.startsWith('image/')" :src="f.url" alt=""><div v-else class="ra-pdf">PDF</div>
          <span>{{ f.name }}</span><button type="button" :aria-label="`Remove ${f.name}`" @click="removePending(f.id)">×</button>
        </div>
      </div>

      <form class="ra-composer" @submit.prevent="send()">
        <input ref="fileEl" type="file" accept=".jpg,.jpeg,.png,.pdf,image/jpeg,image/png,application/pdf" multiple hidden @change="onFileChange">
        <button class="ra-btn ra-attach" type="button" aria-label="Attach floor plan or photo" title="Attach floor plan or photo (JPG, PNG, PDF)" @click="fileEl?.click()">
          <svg viewBox="0 0 24 24"><path d="M21 12.5l-8.6 8.6a5.5 5.5 0 0 1-7.8-7.8l9-9a3.7 3.7 0 0 1 5.2 5.2l-9 9a1.8 1.8 0 0 1-2.6-2.6l8.3-8.3"/></svg>
        </button>
        <textarea ref="inputEl" v-model="draft" rows="1" placeholder="Ask about your property or upload a floor plan…" aria-label="Message"
                  @keydown.enter.exact.prevent="send()" @paste="onPaste"></textarea>
        <button class="ra-btn ra-btn-primary ra-send" type="submit" aria-label="Send" :disabled="busy">
          <svg viewBox="0 0 24 24"><line x1="22" y1="2" x2="11" y2="13"/><polygon points="22 2 15 22 11 13 2 9 22 2"/></svg>
        </button>
      </form>
      <p class="ra-note">AI answers are estimates. Prices, stock and package details are confirmed by Widebed.</p>
    </section>

    <button v-if="!isBig" class="ra-launch" type="button" @click="setOpen(!isOpen)"
            :aria-expanded="String(isOpen)" :aria-label="isOpen ? 'Close chat' : 'Open AI Renovation Assistant'">
      <svg v-if="!isOpen" viewBox="0 0 24 24"><path d="M21 15a2 2 0 0 1-2 2H8l-5 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"/></svg>
      <svg v-else viewBox="0 0 24 24"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>
      <span v-if="unread" class="ra-badge"></span>
    </button>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, onMounted, onBeforeUnmount, defineComponent, h } from 'vue'

const props = defineProps({
  webhookUrl: { type: String, default: 'https://n8n.srv1102332.hstgr.cloud/webhook/reno-assistants' },
  maxFileMB: { type: Number, default: 10 },
  maxFiles: { type: Number, default: 3 },
  historyTurns: { type: Number, default: 10 },
  storageKey: { type: String, default: 'widebed_reno_chat_v1' },
  contextProvider: { type: Function, default: null }
})
const emit = defineEmits(['upload', 'actions', 'message'])
const greeting = 'Hi! I can help you plan your renovation. Upload your floor plan (JPG, PNG or PDF) or ask me a question to begin.'
const defaults = ['Can you analyse my floor plan?', 'Which package suits my property?', 'Can you design my living room?', 'What furniture fits my bedroom?']
const uid = () => 's_' + Date.now().toString(36) + Math.random().toString(36).slice(2, 8)
const isOpen = ref(false), isBig = ref(false), isDragging = ref(false), unread = ref(false), busy = ref(false)
const messages = ref([]), pending = ref([]), uploads = ref([]), suggestions = ref(defaults.slice())
const draft = ref(''), fileEl = ref(null), inputEl = ref(null), logEl = ref(null)
const sessionId = ref(uid()), typingText = ref('Thinking…'), progress = ref(0)
const hasPending = computed(() => pending.value.length > 0)
let abortController = null, progressTimer = null, ctxProvider = props.contextProvider

const ProductCard = defineComponent({
  props: { product: { type: Object, required: true } },
  setup(p) {
    return () => h('div', { class: 'ra-prod' }, [
      /^https?:\/\//i.test(p.product.image || '')
        ? h('a', { href: p.product.image, target: '_blank', rel: 'noopener' }, [h('img', { src: p.product.image, alt: p.product.name, loading: 'lazy', referrerpolicy: 'no-referrer', onError: e => { e.target.style.display = 'none' } })])
        : h('div', { class: 'ra-noimg' }, 'No photo'),
      h('div', { class: 'ra-pn' }, p.product.name),
      h('div', { class: 'ra-pp' }, p.product.price || '')
    ])
  }
})

function formatText(value = '') {
  // Escape all backend/user text before applying the small supported formatting subset.
  const escaped = String(value).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]))
  return escaped.replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>').replace(/^- /gm, '• ').replace(/\n/g, '<br>')
}
function scrollDown() { nextTick(() => { if (logEl.value) logEl.value.scrollTop = logEl.value.scrollHeight }) }
function save() {
  try {
    localStorage.setItem(props.storageKey, JSON.stringify({
      sessionId: sessionId.value,
      messages: messages.value.slice(-60).map(m => ({
        role: m.role, text: m.text, err: !!m.err, products: m.products || [], packages: m.packages || [],
        image: m.image ? { url: m.image.url, ext: m.image.ext } : null,
        files: (m.files || []).map(f => ({ name: f.name, type: f.type }))
      }))
    }))
  } catch {}
}
function load() {
  try {
    const d = JSON.parse(localStorage.getItem(props.storageKey) || 'null')
    if (d && Array.isArray(d.messages)) { sessionId.value = d.sessionId || sessionId.value; messages.value = d.messages }
  } catch {}
}
function notice(text) { messages.value.push({ role: 'bot', text, err: true }); save(); scrollDown() }
function getContext() {
  try {
    if (ctxProvider) return ctxProvider() || {}
    if (window.RoomPlanner?.getDesign) return { room: window.RoomPlanner.getDesign() }
  } catch {}
  return {}
}
function fileKind(file) {
  if (/^image\/(jpeg|png)$/.test(file.type) || (!file.type && /\.(jpe?g|png)$/i.test(file.name))) return 'image'
  if (file.type === 'application/pdf' || (!file.type && /\.pdf$/i.test(file.name))) return 'pdf'
  return ''
}
async function shrinkImage(file) {
  return new Promise(resolve => {
    const url = URL.createObjectURL(file), img = new Image()
    img.onload = () => {
      const max = 2000, k = Math.min(1, max / Math.max(img.width, img.height))
      if (k === 1 && file.size < 2 * 1024 * 1024) { URL.revokeObjectURL(url); resolve(file); return }
      const canvas = document.createElement('canvas')
      canvas.width = Math.round(img.width * k); canvas.height = Math.round(img.height * k)
      const ctx = canvas.getContext('2d')
      ctx.fillStyle = '#fff'; ctx.fillRect(0, 0, canvas.width, canvas.height); ctx.drawImage(img, 0, 0, canvas.width, canvas.height)
      URL.revokeObjectURL(url)
      canvas.toBlob(blob => resolve(blob || file), 'image/jpeg', 0.9)
    }
    img.onerror = () => { URL.revokeObjectURL(url); resolve(null) }
    img.src = url
  })
}
async function addFiles(fileList) {
  for (const file of Array.from(fileList || [])) {
    if (pending.value.length >= props.maxFiles) { notice(`You can attach up to ${props.maxFiles} files per message.`); break }
    const kind = fileKind(file)
    if (!kind || !/\.(jpe?g|png|pdf)$/i.test(file.name)) { notice(`"${file.name}" is not supported. Please upload a JPG, PNG or PDF.`); continue }
    if (file.size > props.maxFileMB * 1024 * 1024) { notice(`"${file.name}" is larger than ${props.maxFileMB} MB.`); continue }
    let blob = file, name = file.name
    if (kind === 'image') {
      blob = await shrinkImage(file)
      if (!blob) { notice(`"${file.name}" could not be read as an image.`); continue }
      if (blob !== file) name = file.name.replace(/\.\w+$/, '') + '.jpg'
    }
    pending.value.push({ id: uid(), name, type: kind === 'pdf' ? 'application/pdf' : (blob.type || 'image/jpeg'), blob, url: kind === 'image' ? URL.createObjectURL(blob) : '' })
  }
}
function removePending(id) {
  const i = pending.value.findIndex(f => f.id === id)
  if (i >= 0) { if (pending.value[i].url) URL.revokeObjectURL(pending.value[i].url); pending.value.splice(i, 1) }
}
function onFileChange(e) { addFiles(e.target.files); e.target.value = '' }
function onDrop(e) { isDragging.value = false; addFiles(e.dataTransfer?.files) }
function onPaste(e) {
  const files = Array.from(e.clipboardData?.files || []).filter(f => f.type.startsWith('image/'))
  if (files.length) { e.preventDefault(); addFiles(files) }
}
async function send(value = draft.value) {
  const text = String(value || '').trim()
  if (busy.value || (!text && !pending.value.length)) return
  if (!props.webhookUrl) { notice('The AI backend is not connected yet. Set webhookUrl to your n8n webhook.'); return }
  const files = pending.value.slice(); pending.value = []; uploads.value.push(...files)
  const userMessage = { role: 'user', text: text || 'Please analyse the attached file.', files: files.map(f => ({ name: f.name, type: f.type, url: f.url })) }
  messages.value.push(userMessage); draft.value = ''
  busy.value = true; typingText.value = files.length ? 'Uploading your file…' : 'Thinking…'; progress.value = 10
  save(); scrollDown()
  if (files.length) emit('upload', { files })
  const history = messages.value.slice(0, -1).filter(m => !m.err).slice(-props.historyTurns * 2)
    .map(m => ({ role: m.role === 'user' ? 'user' : 'assistant', text: m.text }))
  const fd = new FormData()
  fd.append('sessionId', sessionId.value); fd.append('message', userMessage.text)
  fd.append('history', JSON.stringify(history)); fd.append('context', JSON.stringify(getContext()))
  files.forEach((f, i) => fd.append('file' + i, f.blob, f.name))
  const labels = ['Uploading your file…', 'Looking at your photo or plan…', 'Choosing matching Widebed products…', 'Creating your preview picture… this can take up to a minute', 'Almost done…']
  let labelIndex = 0
  progressTimer = setInterval(() => { labelIndex = Math.min(labelIndex + 1, labels.length - 1); typingText.value = labels[labelIndex]; progress.value = Math.min(progress.value + 15, 92) }, 7000)
  abortController = new AbortController()
  const timeout = setTimeout(() => abortController?.abort(), 170000)
  try {
    const res = await fetch(props.webhookUrl, { method: 'POST', body: fd, signal: abortController.signal })
    if (!res.ok) throw new Error('HTTP ' + res.status)
    let data = await res.json()
    if (Array.isArray(data)) data = data[0] || {}
    const reply = typeof data.reply === 'string' && data.reply.trim() ? data.reply : 'Sorry, I could not produce an answer. Please try again.'
    const cleanItems = (items, max) => (Array.isArray(items) ? items : []).slice(0, max).map(p => ({
      name: String(p?.name || '').slice(0, 120), price: String(p?.price || '').slice(0, 40),
      image: /^https?:\/\//i.test(p?.image || '') ? String(p.image) : ''
    })).filter(p => p.name)
    let image = null
    const gi = data.image
    if (gi && /^image\/(png|jpeg|webp)$/.test(gi.mimeType || '') && typeof gi.data === 'string' && /^[A-Za-z0-9+/=\s]+$/.test(gi.data)) {
      try {
        const bin = atob(gi.data), arr = new Uint8Array(bin.length)
        for (let i = 0; i < bin.length; i++) arr[i] = bin.charCodeAt(i)
        image = { url: URL.createObjectURL(new Blob([arr], { type: gi.mimeType })), ext: gi.mimeType.split('/')[1].replace('jpeg', 'jpg') }
      } catch {}
    }
    messages.value.push({ role: 'bot', text: reply, products: cleanItems(data.products, 4), packages: cleanItems(data.packages, 2), image })
    suggestions.value = Array.isArray(data.suggestions) && data.suggestions.length ? data.suggestions.slice(0, 4).map(String) : defaults.slice()
    if (Array.isArray(data.actions) && data.actions.length) emit('actions', { actions: data.actions })
    emit('message', { reply, data })
  } catch (err) {
    messages.value.push({ role: 'bot', err: true, text: err?.name === 'AbortError' ? 'The AI took too long to answer. Please try again, or upload a smaller file.' : `Sorry, I could not reach the AI service (${err?.message || 'network error'}). Please try again.` })
  } finally {
    clearTimeout(timeout); clearInterval(progressTimer); progressTimer = null; abortController = null
    busy.value = false; progress.value = 100; save(); scrollDown()
    if (!isOpen.value) unread.value = true
    nextTick(() => inputEl.value?.focus({ preventScroll: true }))
  }
}
function newChat() {
  if (busy.value) return
  pending.value.forEach(f => f.url && URL.revokeObjectURL(f.url))
  pending.value = []; messages.value = []; sessionId.value = uid(); suggestions.value = defaults.slice(); save()
}
function setOpen(on) { isOpen.value = on; if (!on) isBig.value = false; if (on) { unread.value = false; scrollDown(); nextTick(() => inputEl.value?.focus({ preventScroll: true })) } }
function setBig(on) { isBig.value = on; scrollDown() }
function onKeydown(e) { if (e.key === 'Escape' && isOpen.value) { if (isBig.value) setBig(false); else setOpen(false) } }
function autosize() {
  if (inputEl.value) { inputEl.value.style.height = 'auto'; inputEl.value.style.height = Math.min(inputEl.value.scrollHeight, 120) + 'px' }
}
function getUploads() { return uploads.value.slice() }
function getMessages() { return messages.value.slice() }
function say(text, nextSuggestions) { messages.value.push({ role: 'bot', text: String(text) }); if (nextSuggestions) suggestions.value = nextSuggestions; save(); scrollDown(); if (!isOpen.value) unread.value = true }
defineExpose({ getUploads, getMessages, setContextProvider: fn => { ctxProvider = fn }, say, open: () => setOpen(true), close: () => setOpen(false), expand: () => { setOpen(true); setBig(true) }, toggle: () => setOpen(!isOpen.value), send })

onMounted(() => { load(); scrollDown(); document.addEventListener('keydown', onKeydown); window.RenovationAssistant = {
  getUploads, getMessages, setContextProvider: fn => { ctxProvider = fn }, say,
  open: () => setOpen(true), close: () => setOpen(false), expand: () => { setOpen(true); setBig(true) },
  toggle: () => setOpen(!isOpen.value), send
} })
onBeforeUnmount(() => {
  document.removeEventListener('keydown', onKeydown); clearInterval(progressTimer); abortController?.abort()
  if (window.RenovationAssistant) delete window.RenovationAssistant
})
</script>

<style>
.ra-root {
  --ra-primary: var(--brand-primary, #4A5D45); --ra-primary-hover: var(--brand-primary-hover, #3b4b37);
  --ra-border: var(--brand-border, #E2E0D8); --ra-text: var(--brand-text-main, #2C352B);
  --ra-muted: var(--brand-text-muted, #6C756D); --ra-danger: #B3433B; --ra-bg: #F6F6F3;
  --ra-font: var(--contentfont, 'Plus Jakarta Sans', system-ui, -apple-system, 'Segoe UI', sans-serif);
  --ra-head: var(--headlinefont, 'Playfair Display', Georgia, serif);
  position: fixed; right: 20px; bottom: 20px; z-index: 99990; box-sizing: border-box;
  font-family: var(--ra-font); color: var(--ra-text); line-height: 1.45;
}
.ra-root, .ra-root * { box-sizing: border-box; }
.ra-root button:focus-visible, .ra-root textarea:focus-visible { outline: 2px solid var(--ra-primary); outline-offset: 2px; }
.ra-card { position: absolute; right: 0; bottom: 74px; width: min(390px, calc(100vw - 24px)); display: flex; flex-direction: column; height: min(660px, calc(100vh - 110px)); min-height: 360px; background: #fff; border: 1px solid var(--ra-border); border-radius: 16px; box-shadow: 0 4px 16px rgba(0,0,0,.04); overflow: hidden; }
.ra-head { display: flex; justify-content: space-between; align-items: center; gap: 8px; padding: 14px 16px; border-bottom: 1px solid var(--ra-border); }
.ra-head h3 { margin: 0; font-family: var(--ra-head); font-size: 18px; }
.head-actions { display: flex; gap: 6px; }
.ra-btn { font: inherit; font-size: 13px; font-weight: 600; padding: 8px 12px; border-radius: 10px; border: 1px solid var(--ra-border); background: #fff; color: var(--ra-text); cursor: pointer; display: inline-flex; align-items: center; justify-content: center; gap: 6px; transition: background .2s, border-color .2s; }
.ra-btn:hover { background: #F0EFEA; border-color: var(--ra-primary); }
.ra-btn:disabled { opacity: .5; cursor: not-allowed; }
.ra-btn-primary { background: var(--ra-primary); border-color: var(--ra-primary); color: #fff; }
.ra-btn-primary:hover { background: var(--ra-primary-hover); border-color: var(--ra-primary-hover); }
.ra-log { flex: 1; min-height: 0; overflow-y: auto; padding: 14px 16px; display: flex; flex-direction: column; gap: 10px; background: var(--ra-bg); }
.ra-msg { max-width: 88%; padding: 10px 13px; border-radius: 14px; font-size: 14px; overflow-wrap: anywhere; }
.ra-msg.bot { align-self: flex-start; background: #fff; border: 1px solid var(--ra-border); border-bottom-left-radius: 4px; }
.ra-msg.user { align-self: flex-end; background: var(--ra-primary); color: #fff; border-bottom-right-radius: 4px; }
.ra-msg.err { border-color: var(--ra-danger); color: var(--ra-danger); }
.ra-files, .ra-pending { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 8px; }
.ra-file { display: flex; align-items: center; gap: 6px; max-width: 100%; padding: 4px 8px 4px 4px; border-radius: 8px; background: rgba(0,0,0,.06); font-size: 12px; }
.ra-msg.bot .ra-file { background: #F0EFEA; }
.ra-file img, .ra-file .ra-pdf { width: 40px; height: 40px; border-radius: 6px; object-fit: cover; flex-shrink: 0; }
.ra-pdf { display: flex; align-items: center; justify-content: center; background: #fff; color: var(--ra-danger); font-weight: 700; font-size: 11px; }
.ra-file span { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; max-width: 150px; }
.ra-pending button { all: unset; cursor: pointer; padding: 2px 6px; font-size: 16px; line-height: 1; color: var(--ra-muted); }
.ra-pending button:hover { color: var(--ra-danger); }
.ra-typing { align-self: flex-start; background: #fff; border: 1px solid var(--ra-border); border-radius: 14px; padding: 10px 13px; font-size: 13px; color: var(--ra-muted); }
.ra-dots { display: inline-flex; gap: 4px; margin-right: 8px; vertical-align: middle; }
.ra-dots i { width: 6px; height: 6px; border-radius: 50%; background: var(--ra-muted); animation: ra-b 1s infinite ease-in-out; }
.ra-dots i:nth-child(2) { animation-delay: .15s; } .ra-dots i:nth-child(3) { animation-delay: .3s; }
@keyframes ra-b { 0%, 80%, 100% { opacity: .3; transform: translateY(0); } 40% { opacity: 1; transform: translateY(-3px); } }
.ra-bar { height: 3px; margin-top: 8px; border-radius: 3px; background: var(--ra-border); overflow: hidden; }
.ra-bar b { display: block; height: 100%; background: var(--ra-primary); transition: width .6s ease; }
.ra-suggest { display: flex; gap: 6px; flex-wrap: wrap; padding: 10px 16px 0; border-top: 1px solid var(--ra-border); }
.ra-chip { font: inherit; font-size: 12px; font-weight: 600; padding: 6px 11px; border-radius: 20px; border: 1px solid var(--ra-border); background: #fff; color: var(--ra-muted); cursor: pointer; text-align: left; transition: all .2s; }
.ra-chip:hover { border-color: var(--ra-primary); color: var(--ra-text); }
.ra-composer { display: flex; align-items: flex-end; gap: 8px; padding: 12px 16px 14px; }
.ra-composer textarea { flex: 1; min-width: 0; resize: none; max-height: 120px; padding: 10px 12px; font: inherit; font-size: 14px; color: var(--ra-text); border: 2px solid var(--ra-border); border-radius: 12px; outline: none; transition: border-color .2s; }
.ra-composer textarea:focus { border-color: var(--ra-primary); }
.ra-attach { width: 42px; height: 42px; padding: 0; flex-shrink: 0; }
.ra-attach svg, .ra-send svg, .ra-launch svg, .ra-expand svg { width: 18px; height: 18px; fill: none; stroke: currentColor; stroke-width: 2; stroke-linecap: round; stroke-linejoin: round; }
.ra-send { height: 42px; padding: 0 16px; flex-shrink: 0; }
.ra-note { padding: 0 16px 10px; margin: 0; font-size: 11px; color: var(--ra-muted); }
.ra-drop { outline: 2px dashed var(--ra-primary); outline-offset: -6px; }
.ra-launch { position: absolute; right: 0; bottom: 0; width: 60px; height: 60px; padding: 0; border: none; border-radius: 50%; background: var(--ra-primary); color: #fff; cursor: pointer; display: flex; align-items: center; justify-content: center; box-shadow: 0 8px 24px rgba(0,0,0,.25); transition: transform .2s, background .2s; }
.ra-launch:hover { background: var(--ra-primary-hover); transform: scale(1.06); }
.ra-launch svg { width: 28px; height: 28px; }
.ra-badge { position: absolute; top: 2px; right: 2px; width: 14px; height: 14px; border-radius: 50%; background: #C18C5D; border: 2px solid #fff; }
.ra-x { width: 34px; padding: 8px 0; font-size: 18px; line-height: 1; }
.ra-cards { display: grid; grid-template-columns: repeat(auto-fill, minmax(112px, 1fr)); gap: 8px; margin-top: 10px; }
.ra-prod { border: 1px solid var(--ra-border); border-radius: 10px; overflow: hidden; background: #fff; color: var(--ra-text); }
.ra-prod a { display: block; }
.ra-prod img, .ra-noimg { width: 100%; aspect-ratio: 1 / 1; object-fit: cover; display: block; background: #F0EFEA; }
.ra-noimg { display: flex; align-items: center; justify-content: center; font-size: 11px; color: var(--ra-muted); }
.ra-pn { padding: 6px 8px 0; font-size: 12px; font-weight: 700; line-height: 1.3; overflow-wrap: anywhere; }
.ra-pp { padding: 2px 8px 8px; font-size: 12px; font-weight: 700; color: var(--ra-primary); }
.ra-gen { margin-top: 10px; }
.ra-gen img { display: block; width: 100%; border-radius: 10px; border: 1px solid var(--ra-border); background: #F0EFEA; }
.ra-gen-note { margin-top: 6px; font-size: 11px; color: var(--ra-muted); }
.ra-gen-note a { color: var(--ra-primary); font-weight: 700; margin-left: 6px; }
.ra-cards.pkg { grid-template-columns: repeat(auto-fill, minmax(150px, 1fr)); }
.ra-cards.pkg img, .ra-cards.pkg .ra-noimg { aspect-ratio: 4 / 3; }
.ra-backdrop { position: fixed; inset: 0; background: rgba(30,38,29,.55); }
.ra-root.ra-big .ra-card { position: fixed; top: 50%; left: 50%; right: auto; bottom: auto; transform: translate(-50%, -50%); width: min(980px, 94vw); height: min(90vh, 900px); z-index: 1; box-shadow: 0 20px 60px rgba(0,0,0,.35); }
.ra-root.ra-big .ra-msg { font-size: 15px; max-width: 75%; }
.ra-root.ra-big .ra-cards { grid-template-columns: repeat(auto-fill, minmax(170px, 1fr)); }
.ra-root.ra-big .ra-cards.pkg { grid-template-columns: repeat(auto-fill, minmax(230px, 1fr)); }
@media (max-width: 520px) {
  .ra-root { right: 14px; bottom: 14px; }
  .ra-root.ra-open .ra-card { position: fixed; inset: 0; width: 100%; height: 100%; max-height: none; border-radius: 0; border: none; }
  .ra-root.ra-open .ra-launch { display: none; }
  .ra-expand { display: none; }
}
</style>
