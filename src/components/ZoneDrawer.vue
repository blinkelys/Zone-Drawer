<script setup lang="ts">
import {
  computed,
  onBeforeUnmount,
  onMounted,
  ref,
  watch,
} from 'vue'

interface Pt {
  x: number
  y: number
}

type LabelPosition = 'above' | 'inside'

interface Zone {
  id: string
  name: string
  color: string
  points: Pt[]
  labelPosition: LabelPosition
  labelRotation: number
}

const canvas = ref<HTMLCanvasElement | null>(null)
const wrap = ref<HTMLDivElement | null>(null)
const fileInput = ref<HTMLInputElement | null>(null)

const zones = ref<Zone[]>([])
const tool = ref<'select' | 'draw' | 'rectangle'>('draw')
const selectedId = ref<string | null>(null)
const draft = ref<Pt[]>([])
const newColor = ref('#10b981')

const hover = ref<Pt | null>(null)
const mouseWorld = ref<Pt>({ x: 0, y: 0 })

const undoStack: string[] = []
const redoStack: string[] = []

const STORAGE_KEY = 'zone-drawer'

const selected = computed(
  () => zones.value.find(z => z.id === selectedId.value) ?? null,
)

// -----------------------------------------------------------------------------
// Rectangle drag state
// -----------------------------------------------------------------------------

const rectangleStart = ref<Pt | null>(null)
const rectangleCurrent = ref<Pt | null>(null)
const rectangleDragging = ref(false)

// -----------------------------------------------------------------------------
// Zone moving state
// -----------------------------------------------------------------------------

// Use a ref because movingZone is also accessed by the Vue template.
const movingZone = ref<Zone | null>(null)

let moveStartMouse: Pt | null = null
let moveOriginalPoints: Pt[] | null = null
let moveOriginalState: string | null = null
let moveChanged = false

// -----------------------------------------------------------------------------
// View
// -----------------------------------------------------------------------------

const view = ref({
  x: 0,
  y: 0,
  scale: 40,
})

let size = { w: 0, h: 0 }

let panning: {
  sx: number
  sy: number
  vx: number
  vy: number
} | null = null

let spaceDown = false

function cloneZones(): Zone[] {
  return JSON.parse(JSON.stringify(zones.value)) as Zone[]
}

/**
 * Record the current state before a mutation.
 */
function snapshot() {
  undoStack.push(JSON.stringify(cloneZones()))

  if (undoStack.length > 200) {
    undoStack.shift()
  }

  redoStack.length = 0
}

function restore(json: string) {
  const parsed = JSON.parse(json)

  zones.value = parsed.map((z: Zone) => ({
    ...z,
    labelPosition: z.labelPosition ?? 'above',
    labelRotation: z.labelRotation ?? 0,
    points: z.points.map(p => ({
      x: p.x,
      y: p.y,
    })),
  }))

  if (!zones.value.some(z => z.id === selectedId.value)) {
    selectedId.value = null
  }

  draft.value = []

  rectangleStart.value = null
  rectangleCurrent.value = null
  rectangleDragging.value = false

  movingZone.value = null
  moveStartMouse = null
  moveOriginalPoints = null
  moveOriginalState = null
  moveChanged = false

  draw()
}

function undo() {
  const previous = undoStack.pop()

  if (previous === undefined) return

  redoStack.push(JSON.stringify(cloneZones()))
  restore(previous)
}

function redo() {
  const next = redoStack.pop()

  if (next === undefined) return

  undoStack.push(JSON.stringify(cloneZones()))
  restore(next)
}

function setColor(color: string) {
  if (selected.value) {
    if (selected.value.color === color) return

    snapshot()
    selected.value.color = color
  } else {
    newColor.value = color
  }
}

function setLabelPosition(position: LabelPosition) {
  if (!selected.value) return
  if (selected.value.labelPosition === position) return

  snapshot()
  selected.value.labelPosition = position
}

function rotateLabel() {
  if (!selected.value) return

  snapshot()

  selected.value.labelRotation =
    (selected.value.labelRotation - 90 + 360) % 360
}

function moveSelected(direction: 'forward' | 'backward') {
  if (!selected.value) return

  const index = zones.value.findIndex(
    z => z.id === selected.value!.id,
  )

  if (index === -1) return

  const targetIndex =
    direction === 'forward'
      ? Math.min(index + 1, zones.value.length - 1)
      : Math.max(index - 1, 0)

  if (targetIndex === index) return

  snapshot()

  const [zone] = zones.value.splice(index, 1)

  if (!zone) return

  zones.value.splice(targetIndex, 0, zone)
}

const uid = () => Math.random().toString(36).slice(2, 10)

// -----------------------------------------------------------------------------
// Coordinate conversion
// -----------------------------------------------------------------------------

const toScreen = (p: Pt): Pt => ({
  x: p.x * view.value.scale + view.value.x,
  y: -p.y * view.value.scale + view.value.y,
})

const toWorld = (sx: number, sy: number): Pt => ({
  x: (sx - view.value.x) / view.value.scale,
  y: -(sy - view.value.y) / view.value.scale,
})

// -----------------------------------------------------------------------------
// Snapping
// -----------------------------------------------------------------------------

function snapToGrid(p: Pt): Pt {
  return {
    x: Math.round(p.x),
    y: Math.round(p.y),
  }
}

function constrain(raw: Pt, from: Pt | null): Pt {
  if (!from) {
    return snapToGrid(raw)
  }

  const dx = raw.x - from.x
  const dy = raw.y - from.y

  const adx = Math.abs(dx)
  const ady = Math.abs(dy)

  const sx = Math.sign(dx) || 1
  const sy = Math.sign(dy) || 1

  const angle = Math.atan2(ady, adx)

  if (angle < Math.PI / 8) {
    return {
      x: from.x + sx * Math.round(adx),
      y: from.y,
    }
  }

  if (angle > (3 * Math.PI) / 8) {
    return {
      x: from.x,
      y: from.y + sy * Math.round(ady),
    }
  }

  const d = Math.round((adx + ady) / 2)

  return {
    x: from.x + sx * d,
    y: from.y + sy * d,
  }
}

function snapPoint(sx: number, sy: number): Pt {
  const first = draft.value[0]
  const last = draft.value[draft.value.length - 1] ?? null

  if (first && draft.value.length >= 3) {
    const a = toScreen(first)

    if (Math.hypot(a.x - sx, a.y - sy) < 12) {
      return { ...first }
    }
  }

  return constrain(toWorld(sx, sy), last)
}

// -----------------------------------------------------------------------------
// Zone creation
// -----------------------------------------------------------------------------

function createZone(
  points: Pt[],
  name?: string,
  color = newColor.value,
) {
  if (points.length < 3) return

  snapshot()

  const zone: Zone = {
    id: uid(),
    name: name?.trim() || `Zone ${zones.value.length + 1}`,
    color,
    points: points.map(p => ({ ...p })),
    labelPosition: 'above',
    labelRotation: 0,
  }

  zones.value.push(zone)
  selectedId.value = zone.id
}

function finishDraft() {
  if (draft.value.length < 3) {
    draft.value = []
    return
  }

  const name = window.prompt(
    'Zone name:',
    `Zone ${zones.value.length + 1}`,
  )

  if (name !== null) {
    createZone(draft.value, name)
  }

  draft.value = []
}

function createRectangle(start: Pt, end: Pt) {
  const minX = Math.min(start.x, end.x)
  const maxX = Math.max(start.x, end.x)

  const minY = Math.min(start.y, end.y)
  const maxY = Math.max(start.y, end.y)

  if (maxX - minX < 0.01 || maxY - minY < 0.01) {
    return
  }

  const points: Pt[] = [
    { x: minX, y: minY },
    { x: maxX, y: minY },
    { x: maxX, y: maxY },
    { x: minX, y: maxY },
  ]

  const name = window.prompt(
    'Zone name:',
    `Zone ${zones.value.length + 1}`,
  )

  if (name !== null) {
    createZone(points, name)
  }
}

// -----------------------------------------------------------------------------
// Geometry
// -----------------------------------------------------------------------------

function pointInPoly(p: Pt, poly: Pt[]) {
  let inside = false

  for (
    let i = 0, j = poly.length - 1;
    i < poly.length;
    j = i++
  ) {
    const a = poly[i]!
    const b = poly[j]!

    if (
      a.y > p.y !== b.y > p.y &&
      p.x <
        ((b.x - a.x) * (p.y - a.y)) / (b.y - a.y) +
          a.x
    ) {
      inside = !inside
    }
  }

  return inside
}

function area(poly: Pt[]) {
  let s = 0

  for (let i = 0; i < poly.length; i++) {
    const a = poly[i]!
    const b = poly[(i + 1) % poly.length]!

    s += a.x * b.y - b.x * a.y
  }

  return Math.abs(s) / 2
}

function hitZone(sx: number, sy: number) {
  const w = toWorld(sx, sy)

  return (
    [...zones.value]
      .reverse()
      .find(z => pointInPoly(w, z.points)) ?? null
  )
}

// -----------------------------------------------------------------------------
// Moving zones
// -----------------------------------------------------------------------------

function startMovingZone(z: Zone, mouse: Pt) {
  movingZone.value = z
  moveStartMouse = snapToGrid(mouse)
  moveOriginalPoints = z.points.map(p => ({ ...p }))

  // Preserve the complete state from before the drag.
  // It is only added to the undo stack if the zone actually moves.
  moveOriginalState = JSON.stringify(cloneZones())
  moveChanged = false
}

function updateMovingZone(mouse: Pt) {
  const zone = movingZone.value

  if (!zone || !moveStartMouse || !moveOriginalPoints) {
    return
  }

  const current = snapToGrid(mouse)

  const dx = current.x - moveStartMouse.x
  const dy = current.y - moveStartMouse.y

  if (dx === 0 && dy === 0) {
    return
  }

  zone.points = moveOriginalPoints.map(p => ({
    x: p.x + dx,
    y: p.y + dy,
  }))

  moveChanged = true
}

function finishMovingZone() {
  if (movingZone.value && moveChanged && moveOriginalState) {
    undoStack.push(moveOriginalState)

    if (undoStack.length > 200) {
      undoStack.shift()
    }

    redoStack.length = 0
  }

  movingZone.value = null
  moveStartMouse = null
  moveOriginalPoints = null
  moveOriginalState = null
  moveChanged = false
}

function cancelMovingZone() {
  const zone = movingZone.value

  if (zone && moveOriginalPoints && moveChanged) {
    zone.points = moveOriginalPoints.map(p => ({ ...p }))
  }

  movingZone.value = null
  moveStartMouse = null
  moveOriginalPoints = null
  moveOriginalState = null
  moveChanged = false
}

// -----------------------------------------------------------------------------
// Pointer handling
// -----------------------------------------------------------------------------

function getPointerPosition(e: PointerEvent) {
  const c = canvas.value

  if (!c) {
    return { sx: 0, sy: 0 }
  }

  const r = c.getBoundingClientRect()

  return {
    sx: e.clientX - r.left,
    sy: e.clientY - r.top,
  }
}

function onPointerDown(e: PointerEvent) {
  const c = canvas.value

  if (!c) return

  const { sx, sy } = getPointerPosition(e)
  const world = toWorld(sx, sy)
  const hit = hitZone(sx, sy)

  // Middle mouse / right mouse / space = pan.
  // In select mode, dragging empty space also pans.
  if (
    e.button === 1 ||
    e.button === 2 ||
    spaceDown ||
    (e.button === 0 && tool.value === 'select' && !hit)
  ) {
    panning = {
      sx: e.clientX,
      sy: e.clientY,
      vx: view.value.x,
      vy: view.value.y,
    }

    c.setPointerCapture(e.pointerId)

    if (tool.value === 'select' && e.button === 0) {
      selectedId.value = null
    }

    return
  }

  if (e.button !== 0) return

  // Rectangle tool: both corners snap to the 1 m grid.
  if (tool.value === 'rectangle') {
    rectangleStart.value = snapToGrid(world)
    rectangleCurrent.value = { ...rectangleStart.value }
    rectangleDragging.value = true

    c.setPointerCapture(e.pointerId)
    draw()

    return
  }

  // Polygon drawing.
  if (tool.value === 'draw') {
    const p = snapPoint(sx, sy)
    const first = draft.value[0]

    if (
      first &&
      draft.value.length >= 3 &&
      p.x === first.x &&
      p.y === first.y
    ) {
      finishDraft()
      return
    }

    const last = draft.value[draft.value.length - 1]

    if (last && last.x === p.x && last.y === p.y) {
      return
    }

    draft.value.push(p)

    return
  }

  // Selection and dragging a zone.
  if (tool.value === 'select') {
    selectedId.value = hit?.id ?? null

    if (hit) {
      startMovingZone(hit, world)
      c.setPointerCapture(e.pointerId)
      draw()
    }
  }
}

function onPointerMove(e: PointerEvent) {
  const { sx, sy } = getPointerPosition(e)

  if (panning) {
    view.value.x =
      panning.vx + e.clientX - panning.sx

    view.value.y =
      panning.vy + e.clientY - panning.sy

    draw()
  }

  mouseWorld.value = toWorld(sx, sy)

  if (tool.value === 'draw') {
    hover.value = snapPoint(sx, sy)
  } else {
    hover.value = null
  }

  if (rectangleDragging.value) {
    rectangleCurrent.value = snapToGrid(
      toWorld(sx, sy),
    )

    draw()
  }

  if (movingZone.value) {
    updateMovingZone(toWorld(sx, sy))
    draw()
  }
}

function onPointerUp(e: PointerEvent) {
  const start = rectangleStart.value
  const end = rectangleCurrent.value

  if (
    rectangleDragging.value &&
    start &&
    end
  ) {
    createRectangle(start, end)
  }

  if (movingZone.value) {
    finishMovingZone()
  }

  rectangleStart.value = null
  rectangleCurrent.value = null
  rectangleDragging.value = false
  panning = null

  const c = canvas.value

  if (c?.hasPointerCapture(e.pointerId)) {
    c.releasePointerCapture(e.pointerId)
  }

  draw()
}

// -----------------------------------------------------------------------------
// Zoom / pan
// -----------------------------------------------------------------------------

function onWheel(e: WheelEvent) {
  e.preventDefault()

  const c = canvas.value

  if (!c) return

  const r = c.getBoundingClientRect()

  const sx = e.clientX - r.left
  const sy = e.clientY - r.top

  if (e.ctrlKey || e.metaKey) {
    const before = toWorld(sx, sy)

    view.value.scale = Math.min(
      400,
      Math.max(
        4,
        view.value.scale * Math.exp(-e.deltaY * 0.01),
      ),
    )

    view.value.x = sx - before.x * view.value.scale
    view.value.y = sy + before.y * view.value.scale
  } else {
    view.value.x -= e.deltaX
    view.value.y -= e.deltaY
  }

  draw()
}

// -----------------------------------------------------------------------------
// Zone editing
// -----------------------------------------------------------------------------

function onDblClick(e: MouseEvent) {
  if (tool.value !== 'select') return

  const c = canvas.value

  if (!c) return

  const r = c.getBoundingClientRect()

  const z = hitZone(
    e.clientX - r.left,
    e.clientY - r.top,
  )

  if (z) renameZone(z)
}

function renameZone(z: Zone) {
  const name = window.prompt('Zone name:', z.name)

  if (name !== null) {
    const trimmed = name.trim()

    if (trimmed && trimmed !== z.name) {
      snapshot()
      z.name = trimmed
    }
  }
}

function deleteSelected() {
  if (!selected.value) return

  snapshot()

  zones.value = zones.value.filter(
    z => z.id !== selectedId.value,
  )

  selectedId.value = null
}

// -----------------------------------------------------------------------------
// Keyboard
// -----------------------------------------------------------------------------

function onKey(e: KeyboardEvent) {
  if (e.type === 'keyup') {
    if (e.code === 'Space') {
      spaceDown = false
    }

    return
  }

  if (
    (e.metaKey || e.ctrlKey) &&
    e.key.toLowerCase() === 'z'
  ) {
    e.preventDefault()

    if (e.shiftKey) {
      redo()
    } else {
      undo()
    }

    return
  }

  if (
    (e.metaKey || e.ctrlKey) &&
    e.key.toLowerCase() === 'y'
  ) {
    e.preventDefault()
    redo()
    return
  }

  if (e.code === 'Space') {
    spaceDown = true
    e.preventDefault()
  } else if (e.key === 'Escape') {
    draft.value = []

    rectangleStart.value = null
    rectangleCurrent.value = null
    rectangleDragging.value = false

    cancelMovingZone()
    draw()
  } else if (e.key === 'Enter') {
    if (tool.value === 'draw') {
      finishDraft()
    }
  } else if (
    e.key === 'Backspace' &&
    draft.value.length
  ) {
    draft.value.pop()
  } else if (
    (e.key === 'Delete' || e.key === 'Backspace') &&
    selected.value
  ) {
    deleteSelected()
  } else if (e.key.toLowerCase() === 'v') {
    tool.value = 'select'
  } else if (e.key.toLowerCase() === 'd') {
    tool.value = 'draw'
  } else if (e.key.toLowerCase() === 'r') {
    tool.value = 'rectangle'
  }
}

// -----------------------------------------------------------------------------
// Canvas drawing
// -----------------------------------------------------------------------------

function drawGrid(ctx: CanvasRenderingContext2D) {
  const s = view.value.scale

  const step =
    s >= 12
      ? 1
      : s >= 5
        ? 5
        : 10

  const tl = toWorld(0, 0)
  const br = toWorld(size.w, size.h)

  const color = (v: number) =>
    v % 5 === 0
      ? '#cbd5e1'
      : '#e8edf3'

  for (
    let x = Math.floor(tl.x / step) * step;
    x <= br.x;
    x += step
  ) {
    ctx.strokeStyle = color(x)
    ctx.lineWidth = 1

    const px =
      Math.round(toScreen({ x, y: 0 }).x) + 0.5

    ctx.beginPath()
    ctx.moveTo(px, 0)
    ctx.lineTo(px, size.h)
    ctx.stroke()
  }

  for (
    let y = Math.floor(br.y / step) * step;
    y <= tl.y;
    y += step
  ) {
    ctx.strokeStyle = color(y)
    ctx.lineWidth = 1

    const py =
      Math.round(toScreen({ x: 0, y }).y) + 0.5

    ctx.beginPath()
    ctx.moveTo(0, py)
    ctx.lineTo(size.w, py)
    ctx.stroke()
  }

  ctx.fillStyle = '#94a3b8'
  ctx.font = '10px sans-serif'

  const lab = s >= 20 ? 5 : 10

  for (
    let x = Math.ceil(tl.x / lab) * lab;
    x <= br.x;
    x += lab
  ) {
    ctx.fillText(
      `${x}m`,
      toScreen({ x, y: 0 }).x + 3,
      12,
    )
  }

  for (
    let y = Math.ceil(br.y / lab) * lab;
    y <= tl.y;
    y += lab
  ) {
    ctx.fillText(
      `${y}m`,
      4,
      toScreen({ x: 0, y }).y - 3,
    )
  }
}

function drawZone(
  ctx: CanvasRenderingContext2D,
  z: Zone,
  sel: boolean,
) {
  const pts = z.points.map(toScreen)

  ctx.beginPath()

  pts.forEach((p, i) =>
    i
      ? ctx.lineTo(p.x, p.y)
      : ctx.moveTo(p.x, p.y),
  )

  ctx.closePath()

  ctx.globalAlpha = 0.25
  ctx.fillStyle = z.color
  ctx.fill()

  ctx.globalAlpha = 1
  ctx.strokeStyle = z.color
  ctx.lineWidth = sel ? 3 : 2
  ctx.stroke()

  if (sel) {
    ctx.fillStyle = z.color

    pts.forEach(p => {
      ctx.fillRect(
        p.x - 3,
        p.y - 3,
        6,
        6,
      )
    })
  }

  const xs = pts.map(p => p.x)
  const ys = pts.map(p => p.y)

  const cx = (Math.min(...xs) + Math.max(...xs)) / 2
  const cy = (Math.min(...ys) + Math.max(...ys)) / 2

  ctx.save()

  ctx.font = 'bold 14px sans-serif'
  ctx.fillStyle = '#0f172a'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'

  let labelX = cx
  let labelY = Math.min(...ys) - 16

  if (z.labelPosition === 'inside') {
    labelX = cx
    labelY = cy
  }

  ctx.translate(labelX, labelY)

  if (z.labelRotation) {
    ctx.rotate((z.labelRotation * Math.PI) / 180)
  }

  ctx.fillText(z.name, 0, 0)

  ctx.font = '11px sans-serif'
  ctx.fillStyle = '#475569'
  ctx.fillText(`${area(z.points).toFixed(2)} m²`, 0, 16)

  ctx.restore()
}

function drawStartPoint(
  ctx: CanvasRenderingContext2D,
  point: Pt,
) {
  const start = toScreen(point)

  ctx.save()

  // White outline keeps the blue marker visible on any background.
  ctx.fillStyle = '#ffffff'
  ctx.beginPath()
  ctx.arc(start.x, start.y, 7, 0, Math.PI * 2)
  ctx.fill()

  ctx.fillStyle = '#2563eb'
  ctx.beginPath()
  ctx.arc(start.x, start.y, 4.5, 0, Math.PI * 2)
  ctx.fill()

  ctx.restore()
}

function drawRectanglePreview(ctx: CanvasRenderingContext2D) {
  const start = rectangleStart.value
  const current = rectangleCurrent.value

  if (!rectangleDragging.value || !start || !current) {
    return
  }

  const minX = Math.min(start.x, current.x)
  const maxX = Math.max(start.x, current.x)

  const minY = Math.min(start.y, current.y)
  const maxY = Math.max(start.y, current.y)

  const pts = [
    toScreen({ x: minX, y: minY }),
    toScreen({ x: maxX, y: minY }),
    toScreen({ x: maxX, y: maxY }),
    toScreen({ x: minX, y: maxY }),
  ]

  ctx.save()

  ctx.strokeStyle = '#2563eb'
  ctx.fillStyle = '#2563eb'
  ctx.globalAlpha = 0.15
  ctx.lineWidth = 2

  ctx.beginPath()

  pts.forEach((p, i) =>
    i
      ? ctx.lineTo(p.x, p.y)
      : ctx.moveTo(p.x, p.y),
  )

  ctx.closePath()
  ctx.fill()

  ctx.globalAlpha = 1
  ctx.stroke()

  // Explicitly render the blue starting-point marker.
  drawStartPoint(ctx, start)

  ctx.restore()
}

function draw() {
  const c = canvas.value

  if (!c) return

  const ctx = c.getContext('2d')

  if (!ctx) return

  const dpr = window.devicePixelRatio || 1

  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)

  ctx.fillStyle = '#ffffff'
  ctx.fillRect(0, 0, size.w, size.h)

  drawGrid(ctx)

  for (const z of zones.value) {
    drawZone(ctx, z, z.id === selectedId.value)
  }

  drawRectanglePreview(ctx)

  if (draft.value.length) {
    const pts = draft.value.map(toScreen)

    ctx.strokeStyle = '#2563eb'
    ctx.lineWidth = 2

    ctx.beginPath()

    pts.forEach((p, i) =>
      i
        ? ctx.lineTo(p.x, p.y)
        : ctx.moveTo(p.x, p.y),
    )

    if (hover.value) {
      const h = toScreen(hover.value)
      ctx.lineTo(h.x, h.y)
    }

    ctx.stroke()

    // Draw the first polygon point distinctly as the start position.
    drawStartPoint(ctx, draft.value[0]!)

    ctx.fillStyle = '#2563eb'

    pts.slice(1).forEach(p => {
      ctx.beginPath()
      ctx.arc(p.x, p.y, 4, 0, Math.PI * 2)
      ctx.fill()
    })

    const last = draft.value[draft.value.length - 1]!

    if (
      hover.value &&
      (
        hover.value.x !== last.x ||
        hover.value.y !== last.y
      )
    ) {
      const len = Math.hypot(
        hover.value.x - last.x,
        hover.value.y - last.y,
      )

      const h = toScreen(hover.value)

      ctx.fillStyle = '#1e3a8a'
      ctx.font = '12px sans-serif'
      ctx.fillText(`${len.toFixed(2)} m`, h.x + 10, h.y - 10)
    }
  } else if (hover.value && tool.value === 'draw') {
    const h = toScreen(hover.value)

    ctx.fillStyle = '#2563eb'
    ctx.beginPath()
    ctx.arc(h.x, h.y, 4, 0, Math.PI * 2)
    ctx.fill()
  }
}

// -----------------------------------------------------------------------------
// Resize / storage
// -----------------------------------------------------------------------------

function resize() {
  const c = canvas.value
  const w = wrap.value

  if (!c || !w) return

  const dpr = window.devicePixelRatio || 1

  size = {
    w: w.clientWidth,
    h: w.clientHeight,
  }

  c.width = size.w * dpr
  c.height = size.h * dpr

  c.style.width = `${size.w}px`
  c.style.height = `${size.h}px`

  draw()
}

function exportJson() {
  const data = {
    version: 2,
    unit: 'm',
    zones: zones.value,
  }

  const blob = new Blob(
    [JSON.stringify(data, null, 2)],
    { type: 'application/json' },
  )

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

    if (!Array.isArray(list)) {
      throw new Error('missing zones')
    }

    const next: Zone[] = list.map(
      (
        z: {
          name?: unknown
          color?: unknown
          points?: Pt[]
          labelPosition?: unknown
          labelRotation?: unknown
        },
      ) => {
        if (
          !Array.isArray(z.points) ||
          z.points.some(
            p =>
              typeof p.x !== 'number' ||
              typeof p.y !== 'number',
          )
        ) {
          throw new Error('bad zone')
        }

        return {
          id: uid(),
          name: String(z.name ?? 'Zone'),
          color:
            typeof z.color === 'string' &&
            /^#[0-9a-f]{6}$/i.test(z.color)
              ? z.color
              : '#10b981',
          points: z.points.map(p => ({
            x: p.x,
            y: p.y,
          })),
          labelPosition:
            z.labelPosition === 'inside' ? 'inside' : 'above',
          labelRotation:
            typeof z.labelRotation === 'number'
              ? z.labelRotation
              : 0,
        }
      },
    )

    snapshot()

    zones.value = next
    selectedId.value = null
    draft.value = []
  } catch {
    window.alert('Could not import: invalid zones file.')
  }
}

function clearAll() {
  if (
    zones.value.length &&
    window.confirm('Remove all zones?')
  ) {
    snapshot()
    zones.value = []
    selectedId.value = null
  }
}

function resetView() {
  view.value = {
    x: size.w / 2,
    y: size.h / 2,
    scale: 40,
  }
}

// -----------------------------------------------------------------------------
// Lifecycle
// -----------------------------------------------------------------------------

onMounted(() => {
  try {
    const saved = JSON.parse(
      localStorage.getItem(STORAGE_KEY) ?? '[]',
    )

    if (Array.isArray(saved)) {
      zones.value = saved.map((z: Zone) => ({
        ...z,
        color: z.color ?? '#10b981',
        labelPosition: z.labelPosition ?? 'above',
        labelRotation: z.labelRotation ?? 0,
      }))
    }
  } catch {
    // Ignore corrupt storage.
  }

  resize()
  resetView()

  window.addEventListener('resize', resize)
  window.addEventListener('keydown', onKey)
  window.addEventListener('keyup', onKey)

  canvas.value?.addEventListener('wheel', onWheel, {
    passive: false,
  })
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', resize)
  window.removeEventListener('keydown', onKey)
  window.removeEventListener('keyup', onKey)
  canvas.value?.removeEventListener('wheel', onWheel)
})

watch(
  [
    zones,
    draft,
    hover,
    selectedId,
    view,
    tool,
    rectangleStart,
    rectangleCurrent,
    rectangleDragging,
    movingZone,
  ],
  draw,
  { deep: true },
)

watch(
  zones,
  z => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(z))
  },
  { deep: true },
)
</script>

<template>
  <div class="h-screen flex flex-col bg-slate-100 text-slate-800">
    <nav
      class="flex items-center gap-2 px-4 py-2 bg-slate-900 text-white text-sm shrink-0"
    >
      <span class="font-semibold mr-4">
        Zone Drawer
      </span>

      <button
        class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600"
        @click="exportJson"
      >
        Save
      </button>

      <button
        class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600"
        @click="fileInput?.click()"
      >
        Import
      </button>

      <input
        ref="fileInput"
        type="file"
        accept="application/json,.json"
        class="hidden"
        @change="importJson"
      />

      <span class="mx-2 h-5 w-px bg-slate-600" />

      <button
        class="px-3 py-1 rounded"
        :class="
          tool === 'draw'
            ? 'bg-blue-600'
            : 'bg-slate-700 hover:bg-slate-600'
        "
        @click="tool = 'draw'"
      >
        Draw zone (D)
      </button>

      <button
        class="px-3 py-1 rounded"
        :class="
          tool === 'rectangle'
            ? 'bg-blue-600'
            : 'bg-slate-700 hover:bg-slate-600'
        "
        @click="tool = 'rectangle'"
      >
        Rectangle (R)
      </button>

      <button
        class="px-3 py-1 rounded"
        :class="
          tool === 'select'
            ? 'bg-blue-600'
            : 'bg-slate-700 hover:bg-slate-600'
        "
        @click="tool = 'select'"
      >
        Select (V)
      </button>

      <button
        v-if="draft.length >= 3"
        class="px-3 py-1 rounded bg-emerald-600 hover:bg-emerald-500"
        @click="finishDraft"
      >
        Finish zone
      </button>

      <template v-if="selected">
        <button
          class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600"
          @click="renameZone(selected)"
        >
          Rename
        </button>

        <button
          class="px-3 py-1 rounded bg-red-600 hover:bg-red-500"
          @click="deleteSelected"
        >
          Delete
        </button>

        <span class="mx-1 h-5 w-px bg-slate-600" />

        <button
          class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600 disabled:opacity-40"
          :disabled="
            zones.findIndex(z => z.id === selected?.id) >=
            zones.length - 1
          "
          title="Move zone one layer forward"
          @click="moveSelected('forward')"
        >
          Forward ↑
        </button>

        <button
          class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600 disabled:opacity-40"
          :disabled="
            zones.findIndex(z => z.id === selected?.id) <= 0
          "
          title="Move zone one layer backward"
          @click="moveSelected('backward')"
        >
          Backward ↓
        </button>

        <span class="mx-1 h-5 w-px bg-slate-600" />

        <button
          class="px-3 py-1 rounded"
          :class="
            selected.labelPosition === 'above'
              ? 'bg-blue-600'
              : 'bg-slate-700 hover:bg-slate-600'
          "
          @click="setLabelPosition('above')"
        >
          Text above
        </button>

        <button
          class="px-3 py-1 rounded"
          :class="
            selected.labelPosition === 'inside'
              ? 'bg-blue-600'
              : 'bg-slate-700 hover:bg-slate-600'
          "
          @click="setLabelPosition('inside')"
        >
          Text inside
        </button>

        <button
          class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600"
          title="Rotate text 90° counter-clockwise"
          @click="rotateLabel"
        >
          ↶ 90°
        </button>
      </template>

      <label
        class="flex items-center gap-1 ml-2"
        :title="
          selected
            ? 'Color of selected zone'
            : 'Color for new zones'
        "
      >
        Color

        <input
          type="color"
          class="h-7 w-9 rounded cursor-pointer bg-transparent"
          :value="selected ? selected.color : newColor"
          @input="
            setColor(
              ($event.target as HTMLInputElement).value
            )
          "
        />
      </label>

      <button
        class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600 disabled:opacity-40"
        :disabled="undoStack.length === 0"
        @click="undo"
      >
        Undo
      </button>

      <button
        class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600 disabled:opacity-40"
        :disabled="redoStack.length === 0"
        @click="redo"
      >
        Redo
      </button>

      <span class="flex-1" />

      <button
        class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600"
        @click="resetView"
      >
        Reset view
      </button>

      <button
        class="px-3 py-1 rounded bg-slate-700 hover:bg-slate-600"
        @click="clearAll"
      >
        Clear
      </button>
    </nav>

    <div
      ref="wrap"
      class="flex-1 relative overflow-hidden"
    >
      <canvas
        ref="canvas"
        class="absolute inset-0"
        :class="
          tool === 'draw'
            ? 'cursor-crosshair'
            : tool === 'rectangle'
              ? 'cursor-crosshair'
              : movingZone
                ? 'cursor-move'
                : 'cursor-default'
        "
        @pointerdown="onPointerDown"
        @pointermove="onPointerMove"
        @pointerup="onPointerUp"
        @pointercancel="onPointerUp"
        @dblclick="onDblClick"
        @contextmenu.prevent
      />

      <div
        class="absolute bottom-2 left-2 text-xs bg-white/90 rounded px-2 py-1 shadow pointer-events-none"
      >
        x {{ mouseWorld.x.toFixed(1) }} m,
        y {{ mouseWorld.y.toFixed(1) }} m ·
        grid 1 m ·

        <span v-if="tool === 'draw'">
          click to add points, click first
          point or Enter to close
        </span>

        <span v-else-if="tool === 'rectangle'">
          click and drag from one corner to
          the opposite corner
        </span>

        <span v-else>
          click and drag a zone to move it,
          double-click to rename
        </span>

        · Backspace removes last point,
        Esc cancel, Ctrl+Z undo ·
        pan: scroll / right-drag / Space+drag ·
        zoom: pinch or Ctrl+scroll
      </div>
    </div>
  </div>
</template>