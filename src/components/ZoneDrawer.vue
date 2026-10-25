<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'

interface Pt { x: number; y: number }
interface Zone { id: string; name: string; color: string; points: Pt[] }

const canvas = ref<HTMLCanvasElement | null>(null)
const wrap = ref<HTMLDivElement | null>(null)
const fileInput = ref<HTMLInputElement | null>(null)

const zones = ref<Zone[]>([])
const tool = ref<'select' | 'draw'>('draw')
const selectedId = ref<string | null>(null)
const draft = ref<Pt[]>([])
const newColor = ref('#10b981')
const undoStack: string[] = []
const redoStack: string[] = []

function snapshot() {
  undoStack.push(JSON.stringify(zones.value))
  if (undoStack.length > 200) undoStack.shift()
  redoStack.length = 0
}
function restore(json: string) {
  zones.value = JSON.parse(json)
  if (!zones.value.some(z => z.id === selectedId.value)) selectedId.value = null
}
function undo() {
  const prev = undoStack.pop()
  if (prev === undefined) return
  redoStack.push(JSON.stringify(zones.value))
  restore(prev)
}
function redo() {
  const next = redoStack.pop()
  if (next === undefined) return
  undoStack.push(JSON.stringify(zones.value))
  restore(next)
}
function setColor(color: string) {
  if (selected.value) {
    if (selected.value.color !== color) snapshot()
    selected.value.color = color
  } else newColor.value = color
}
const hover = ref<Pt | null>(null)
const mouseWorld = ref<Pt>({ x: 0, y: 0 })

// view.scale = pixels per metre
const view = ref({ x: 0, y: 0, scale: 40 })
let size = { w: 0, h: 0 }
let panning: { sx: number; sy: number; vx: number; vy: number } | null = null
let spaceDown = false

const STORAGE_KEY = 'zone-drawer'
const selected = computed(() => zones.value.find(z => z.id === selectedId.value) ?? null)

const toScreen = (p: Pt): Pt => ({ x: p.x * view.value.scale + view.value.x, y: -p.y * view.value.scale + view.value.y })
const toWorld = (sx: number, sy: number): Pt => ({ x: (sx - view.value.x) / view.value.scale, y: -(sy - view.value.y) / view.value.scale })

// Constrain next point to horizontal, vertical or 45° diagonal on whole metres.
function constrain(raw: Pt, from: Pt | null): Pt {
  if (!from) return { x: Math.round(raw.x), y: Math.round(raw.y) }
  const dx = raw.x - from.x, dy = raw.y - from.y
  const adx = Math.abs(dx), ady = Math.abs(dy)
  const sx = Math.sign(dx) || 1, sy = Math.sign(dy) || 1
  const angle = Math.atan2(ady, adx)
  if (angle < Math.PI / 8) return { x: from.x + sx * Math.round(adx), y: from.y }
  if (angle > (3 * Math.PI) / 8) return { x: from.x, y: from.y + sy * Math.round(ady) }
  const d = Math.round((adx + ady) / 2)
  return { x: from.x + sx * d, y: from.y + sy * d }
}

function snapPoint(sx: number, sy: number): Pt {
  const first = draft.value[0]
  const last = draft.value[draft.value.length - 1] ?? null
  // allow closing on the first point when the cursor is near it
  if (first && draft.value.length >= 3) {
    const a = toScreen(first)
    if (Math.hypot(a.x - sx, a.y - sy) < 12) return { ...first }
  }
  return constrain(toWorld(sx, sy), last)
}

const uid = () => Math.random().toString(36).slice(2, 10)

function finishDraft() {
  if (draft.value.length >= 3) {
    const name = window.prompt('Zone name:', `Zone ${zones.value.length + 1}`)
    if (name !== null) {
      snapshot()
      const z = { id: uid(), name: name.trim() || `Zone ${zones.value.length + 1}`, color: newColor.value, points: [...draft.value] }
      zones.value.push(z)
      selectedId.value = z.id
    }
  }
  draft.value = []
}

function pointInPoly(p: Pt, poly: Pt[]) {
  let inside = false
  for (let i = 0, j = poly.length - 1; i < poly.length; j = i++) {
    const a = poly[i]!, b = poly[j]!
    if (a.y > p.y !== b.y > p.y && p.x < ((b.x - a.x) * (p.y - a.y)) / (b.y - a.y) + a.x) inside = !inside
  }
  return inside
}

function area(poly: Pt[]) {
  let s = 0
  for (let i = 0; i < poly.length; i++) {
    const a = poly[i]!, b = poly[(i + 1) % poly.length]!
    s += a.x * b.y - b.x * a.y
  }
  return Math.abs(s) / 2
}

function hitZone(sx: number, sy: number) {
  const w = toWorld(sx, sy)
  return [...zones.value].reverse().find(z => pointInPoly(w, z.points)) ?? null
}

function onPointerDown(e: PointerEvent) {
  const r = canvas.value!.getBoundingClientRect()
  const sx = e.clientX - r.left, sy = e.clientY - r.top
  const hit = hitZone(sx, sy)
  if (e.button === 1 || e.button === 2 || spaceDown || (e.button === 0 && tool.value === 'select' && !hit)) {
    panning = { sx: e.clientX, sy: e.clientY, vx: view.value.x, vy: view.value.y }
    canvas.value!.setPointerCapture(e.pointerId)
    if (tool.value === 'select' && e.button === 0) selectedId.value = null
    return
  }
  if (e.button !== 0) return
  if (tool.value === 'draw') {
    const p = snapPoint(sx, sy)
    const first = draft.value[0]
    if (first && draft.value.length >= 3 && p.x === first.x && p.y === first.y) return finishDraft()
    const last = draft.value[draft.value.length - 1]
    if (last && last.x === p.x && last.y === p.y) return
    draft.value.push(p)
  } else {
    selectedId.value = hit?.id ?? null
  }
}

function onPointerMove(e: PointerEvent) {
  const r = canvas.value!.getBoundingClientRect()
  const sx = e.clientX - r.left, sy = e.clientY - r.top
  if (panning) {
    view.value.x = panning.vx + e.clientX - panning.sx
    view.value.y = panning.vy + e.clientY - panning.sy
  }
  mouseWorld.value = toWorld(sx, sy)
  hover.value = tool.value === 'draw' ? snapPoint(sx, sy) : null
}

function onPointerUp() { panning = null }

// ctrl/pinch zooms, plain scroll pans
function onWheel(e: WheelEvent) {
  e.preventDefault()
  const r = canvas.value!.getBoundingClientRect()
  const sx = e.clientX - r.left, sy = e.clientY - r.top
  if (e.ctrlKey || e.metaKey) {
    const before = toWorld(sx, sy)
    view.value.scale = Math.min(400, Math.max(4, view.value.scale * Math.exp(-e.deltaY * 0.01)))
    view.value.x = sx - before.x * view.value.scale
    view.value.y = sy + before.y * view.value.scale
  } else {
    view.value.x -= e.deltaX
    view.value.y -= e.deltaY
  }
}

function onDblClick(e: MouseEvent) {
  if (tool.value !== 'select') return
  const r = canvas.value!.getBoundingClientRect()
  const z = hitZone(e.clientX - r.left, e.clientY - r.top)
  if (z) renameZone(z)
}

function renameZone(z: Zone) {
  const name = window.prompt('Zone name:', z.name)
  if (name !== null) { snapshot(); z.name = name.trim() || z.name }
}

function deleteSelected() {
  if (!selected.value) return
  snapshot()
  zones.value = zones.value.filter(z => z.id !== selectedId.value)
  selectedId.value = null
}

function onKey(e: KeyboardEvent) {
  if (e.type === 'keyup') { if (e.code === 'Space') spaceDown = false; return }
  if ((e.metaKey || e.ctrlKey) && e.key.toLowerCase() === 'z') {
    e.preventDefault()
    if (e.shiftKey) redo(); else undo()
    return
  }
  if ((e.metaKey || e.ctrlKey) && e.key.toLowerCase() === 'y') { e.preventDefault(); redo(); return }
  if (e.code === 'Space') { spaceDown = true; e.preventDefault() }
  else if (e.key === 'Escape') draft.value = []
  else if (e.key === 'Enter') finishDraft()
  else if (e.key === 'Backspace' && draft.value.length) draft.value.pop()
  else if ((e.key === 'Delete' || e.key === 'Backspace') && selected.value) deleteSelected()
  else if (e.key === 'v') tool.value = 'select'
  else if (e.key === 'd') tool.value = 'draw'
}

function drawGrid(ctx: CanvasRenderingContext2D) {
  const s = view.value.scale
  const step = s >= 12 ? 1 : s >= 5 ? 5 : 10
  const tl = toWorld(0, 0), br = toWorld(size.w, size.h)
  const color = (v: number) => (v % 5 === 0 ? '#cbd5e1' : '#e8edf3')
  for (let x = Math.floor(tl.x / step) * step; x <= br.x; x += step) {
    ctx.strokeStyle = color(x)
    ctx.lineWidth = 1
    const px = Math.round(toScreen({ x, y: 0 }).x) + 0.5
    ctx.beginPath(); ctx.moveTo(px, 0); ctx.lineTo(px, size.h); ctx.stroke()
  }
  for (let y = Math.floor(br.y / step) * step; y <= tl.y; y += step) {
    ctx.strokeStyle = color(y)
    ctx.lineWidth = 1
    const py = Math.round(toScreen({ x: 0, y }).y) + 0.5
    ctx.beginPath(); ctx.moveTo(0, py); ctx.lineTo(size.w, py); ctx.stroke()
  }
  ctx.fillStyle = '#94a3b8'
  ctx.font = '10px sans-serif'
  const lab = s >= 20 ? 5 : 10
  for (let x = Math.ceil(tl.x / lab) * lab; x <= br.x; x += lab) {
    ctx.fillText(`${x}m`, toScreen({ x, y: 0 }).x + 3, 12)
  }
  for (let y = Math.ceil(br.y / lab) * lab; y <= tl.y; y += lab) {
    ctx.fillText(`${y}m`, 4, toScreen({ x: 0, y }).y - 3)
  }
}

function drawZone(ctx: CanvasRenderingContext2D, z: Zone, sel: boolean) {
  const pts = z.points.map(toScreen)
  ctx.beginPath()
  pts.forEach((p, i) => (i ? ctx.lineTo(p.x, p.y) : ctx.moveTo(p.x, p.y)))
  ctx.closePath()
  ctx.globalAlpha = 0.25
  ctx.fillStyle = z.color
  ctx.fill()
  ctx.globalAlpha = 1
  ctx.strokeStyle = z.color
  ctx.lineWidth = sel ? 3 : 2
  ctx.stroke()
  if (sel) { ctx.fillStyle = z.color; pts.forEach(p => ctx.fillRect(p.x - 3, p.y - 3, 6, 6)) }
  const ys = pts.map(p => p.y), xs = pts.map(p => p.x)
  const cx = (Math.min(...xs) + Math.max(...xs)) / 2
  ctx.textAlign = 'center'
  ctx.font = 'bold 14px sans-serif'
  ctx.fillStyle = '#0f172a'
  ctx.fillText(z.name, cx, Math.min(...ys) - 8) // name above the zone
  ctx.font = '11px sans-serif'
  ctx.fillStyle = '#475569'
  ctx.fillText(`${+area(z.points).toFixed(2)} m²`, cx, (Math.min(...ys) + Math.max(...ys)) / 2)
  ctx.textAlign = 'start'
}

function draw() {
  const c = canvas.value
  if (!c) return
  const ctx = c.getContext('2d')!
  const dpr = window.devicePixelRatio || 1
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.fillStyle = '#ffffff'
  ctx.fillRect(0, 0, size.w, size.h)
  drawGrid(ctx)
  for (const z of zones.value) drawZone(ctx, z, z.id === selectedId.value)

  if (draft.value.length) {
    const pts = draft.value.map(toScreen)
    ctx.strokeStyle = '#2563eb'
    ctx.lineWidth = 2
    ctx.beginPath()
    pts.forEach((p, i) => (i ? ctx.lineTo(p.x, p.y) : ctx.moveTo(p.x, p.y)))
    if (hover.value) { const h = toScreen(hover.value); ctx.lineTo(h.x, h.y) }
    ctx.stroke()
    ctx.fillStyle = '#2563eb'
    pts.forEach(p => { ctx.beginPath(); ctx.arc(p.x, p.y, 4, 0, Math.PI * 2); ctx.fill() })
    const last = draft.value[draft.value.length - 1]!
    if (hover.value && (hover.value.x !== last.x || hover.value.y !== last.y)) {
      const len = Math.hypot(hover.value.x - last.x, hover.value.y - last.y)
      const h = toScreen(hover.value)
      ctx.fillStyle = '#1e3a8a'
      ctx.font = '12px sans-serif'
      ctx.fillText(`${+len.toFixed(2)} m`, h.x + 10, h.y - 10)
    }
  } else if (hover.value && tool.value === 'draw') {
    const h = toScreen(hover.value)
    ctx.fillStyle = '#2563eb'
    ctx.beginPath(); ctx.arc(h.x, h.y, 4, 0, Math.PI * 2); ctx.fill()
  }
}

function resize() {
  const c = canvas.value, w = wrap.value
  if (!c || !w) return
  const dpr = window.devicePixelRatio || 1
  size = { w: w.clientWidth, h: w.clientHeight }
  c.width = size.w * dpr
  c.height = size.h * dpr
  c.style.width = size.w + 'px'
  c.style.height = size.h + 'px'
  draw()
}

function exportJson() {
  const data = { version: 1, unit: 'm', zones: zones.value }
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  const a = document.createElement('a')
  a.href = URL.createObjectURL(blob)
  a.download = 'zones.json'
  a.click()
  URL.revokeObjectURL(a.href)
}

async function importJson(e: Event) {
  const input = e.target as HTMLInputElement
  const file = input.files?.[0]
  input.value = ''
  if (!file) return
  try {
    const data = JSON.parse(await file.text())
    const list = Array.isArray(data) ? data : data.zones
    if (!Array.isArray(list)) throw new Error('missing zones')
    const next: Zone[] = list.map((z: { name?: unknown; color?: unknown; points?: Pt[] }) => {
      if (!Array.isArray(z.points) || z.points.some(p => typeof p.x !== 'number' || typeof p.y !== 'number')) throw new Error('bad zone')
      return { id: uid(), name: String(z.name ?? 'Zone'), color: typeof z.color === 'string' && /^#[0-9a-f]{6}$/i.test(z.color) ? z.color : '#10b981', points: z.points.map(p => ({ x: p.x, y: p.y })) }
    })
    snapshot()
    zones.value = next
    selectedId.value = null
    draft.value = []
  } catch {
    window.alert('Could not import: invalid zones file.')
  }
}

function clearAll() {
  if (zones.value.length && window.confirm('Remove all zones?')) { snapshot(); zones.value = []; selectedId.value = null }
}

function resetView() { view.value = { x: size.w / 2, y: size.h / 2, scale: 40 } }

onMounted(() => {
  try {
    const saved = JSON.parse(localStorage.getItem(STORAGE_KEY) ?? '[]')
    if (Array.isArray(saved)) zones.value = saved.map((z: Zone) => ({ ...z, color: z.color ?? '#10b981' }))
  } catch { /* ignore corrupt storage */ }
  resize()
  resetView()
  window.addEventListener('resize', resize)
  window.addEventListener('keydown', onKey)
  window.addEventListener('keyup', onKey)
  canvas.value!.addEventListener('wheel', onWheel, { passive: false })
})
onBeforeUnmount(() => {
  window.removeEventListener('resize', resize)
  window.removeEventListener('keydown', onKey)
  window.removeEventListener('keyup', onKey)
})
watch([zones, draft, hover, selectedId, view, tool], draw, { deep: true })
watch(zones, z => localStorage.setItem(STORAGE_KEY, JSON.stringify(z)), { deep: true })
</script>

<template>
  <div class="h-screen flex flex-col bg-slate-100 text-slate-800">
    <nav class="flex items-center gap-2 px-4 py-2 bg-slate-900 text-white text-sm shrink-0">
      <span class="font-semibold mr-4">Zone Drawer</span>
      <button class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600" @click="exportJson">Save</button>
      <button class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600" @click="fileInput?.click()">Import</button>
      <input ref="fileInput" type="file" accept="application/json,.json" class="hidden" @change="importJson" />
      <span class="mx-2 h-5 w-px bg-slate-600" />
      <button class="px-3 py-1 rounded" :class="tool === 'draw' ? 'bg-blue-600' : 'bg-slate-700 hover:bg-slate-600'" @click="tool = 'draw'">Draw zone (D)</button>
      <button class="px-3 py-1 rounded" :class="tool === 'select' ? 'bg-blue-600' : 'bg-slate-700 hover:bg-slate-600'" @click="tool = 'select'">Select (V)</button>
      <button v-if="draft.length >= 3" class="px-3 py-1 rounded bg-emerald-600 hover:bg-emerald-500" @click="finishDraft">Finish zone</button>
      <template v-if="selected">
        <button class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600" @click="renameZone(selected)">Rename</button>
        <button class="px-3 py-1 rounded bg-red-600 hover:bg-red-500" @click="deleteSelected">Delete</button>
      </template>
      <label class="flex items-center gap-1 ml-2" :title="selected ? 'Color of selected zone' : 'Color for new zones'">
        Color
        <input type="color" class="h-7 w-9 rounded cursor-pointer bg-transparent" :value="selected ? selected.color : newColor" @input="setColor(($event.target as HTMLInputElement).value)" />
      </label>
      <button class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600" @click="undo">Undo</button>
      <button class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600" @click="redo">Redo</button>
      <span class="flex-1" />
      <button class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600" @click="resetView">Reset view</button>
      <button class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600" @click="clearAll">Clear</button>
    </nav>
    <div ref="wrap" class="flex-1 relative overflow-hidden">
      <canvas
        ref="canvas"
        class="absolute inset-0"
        :class="tool === 'draw' ? 'cursor-crosshair' : 'cursor-default'"
        @pointerdown="onPointerDown"
        @pointermove="onPointerMove"
        @pointerup="onPointerUp"
        @dblclick="onDblClick"
        @contextmenu.prevent
      />
      <div class="absolute bottom-2 left-2 text-xs bg-white/90 rounded px-2 py-1 shadow pointer-events-none">
        x {{ mouseWorld.x.toFixed(1) }} m, y {{ mouseWorld.y.toFixed(1) }} m · grid 1 m ·
        click to add points (snaps to straight/45° lines), click first point or Enter to close, Backspace removes last point, Esc cancel, Ctrl+Z undo ·
        pan: scroll / right-drag / Space+drag · zoom: pinch or Ctrl+scroll
      </div>
    </div>
  </div>
</template>
