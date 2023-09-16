<script setup lang='ts'>
import { ref, onMounted, onUnmounted } from 'vue'
import Konva from 'konva'
import { nanoid } from 'nanoid'


let windowWidth: number = window.innerWidth
let windowHeight: number = window.innerHeight / 2

let CanvasResizeTimeOut: NodeJS.Timeout
const UpdateKonvaCanvasSize = () => {
  const toolbar: HTMLDivElement | null = document.querySelector('#toolbar')
  if (toolbar == null) { return }
  windowWidth = window.innerWidth
  windowHeight = window.innerHeight - toolbar.offsetHeight
  // update Konva Canavs size
  stage.value?.width(windowWidth)
  stage.value?.height(windowHeight)
}
const handleWindowResize = () => {
  clearTimeout(CanvasResizeTimeOut)
  CanvasResizeTimeOut = setTimeout(UpdateKonvaCanvasSize, 300)
}

function loadRefImageFromDataTransferItem(item: DataTransferItem) {
  if (item.type.startsWith('image')) {
    // get the pasted item as a blob
    const blob = item.getAsFile()
    if (blob == null) { return }
    const reader = new FileReader()
    const abspointer = stage.value?.getPointerPosition()
    console.log('abspointer')
    console.log(abspointer)
    const pointer = stage.value?.getRelativePointerPosition()
    reader.onload = () => {
      // create a new RefImage into the current canvas
      RefImage.fromURL(reader.result as string, (img) => {
        // If the abspointer is not in the center 80% of the screen,
        // then maybe the pointer is not correct, 
        // it's the location when the cursor moved out from the screen
        // we need to set it invalid

        // Move the Image so that the center of the image is at the pointer
        console.log('pointer')
        console.log(pointer)
        if (pointer != undefined && document.hasFocus()) {
          img.x(pointer.x)
          img.y(pointer.y)
        } else if (stage.value != null) {
          console.log('fallback to the center')
          img.x(((windowWidth / 2) / stage.value.scaleX() - stage.value.x()))
          img.y(((windowHeight / 2) / stage.value.scaleY() - stage.value.y()))
        }
        flushBackSelectedImgs()
        layers[active_canvas_id.value].add(img)
      })
    }
    reader.readAsDataURL(blob)
  }
}

const preventDefault = (e: Event) => { e.preventDefault() }

const handlePaste = (pasteEvent: any) => {
  console.log('paste')
  for (const element of pasteEvent.clipboardData.items) {
    loadRefImageFromDataTransferItem(element)
  }
}

const handleDropOver = (dropEvent: any) => {
  console.log('dropover')
  dropEvent.preventDefault()
  for (const element of dropEvent.dataTransfer.items) {
    loadRefImageFromDataTransferItem(element)
  }
}

onMounted(() => {
  initKonva()
  bindDragDrop()
  document.addEventListener('paste', handlePaste, false)
  window.addEventListener('keydown', handleStageKeyDown, false)
  window.addEventListener('resize', handleWindowResize, false)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleWindowResize)
  window.removeEventListener('keydown', handleStageKeyDown)
  document.removeEventListener('paste', handlePaste)
  unbindDragDrop()
})


const transformerConfig = {
  anchorStroke: 'white',
  anchorFill: '#168e99',
  anchorSize: 24,
  anchorCornerRadius: 12,
  borderStroke: '#168e99',
  borderStrokeWidth: 4,
  borderDash: [1, 0],
  keepratio: true,
  shouldOverdrawWholeArea: true,
}
const transformer = new Konva.Transformer(transformerConfig)

class RefImage extends Konva.Image {
  constructor(data: HTMLImageElement) {
    super({
      draggable: false,
      image: data,
    })
    this.id(nanoid())
    data.onload = () => {
      this.offsetX(data.width / 2)
      this.offsetY(data.height / 2)
    }
  }
  static fromURL(url: string, callback: (img: Konva.Image) => void, onError?: OnErrorEventHandler | undefined): void {
    Konva.Image.fromURL(url, (img: Konva.Image) => {
      img.id(nanoid())
      img.offsetX(img.width() / 2)
      img.offsetY(img.height() / 2)
      callback(img)
    }, onerror)
  }
}

class RefLayer extends Konva.Layer {
  layer_index: number
  constructor(id: number) {
    super()
    this.layer_index = id
    this.addName('refLayer')
    this.draggable(false)
  }
}

const active_canvas_id = ref(0)
const layers = Array.from(Array(6), (v, k) => (new RefLayer(k)))
const select_layer = new RefLayer(999)
// konva elements
const stage = ref<Konva.Stage | null>(null)
let isMovingCanvas = false

const menu_items = [
  { title: 'Home', icon: 'mdi-home' },
  { title: 'About', icon: 'mdi-help-box' },
  { title: 'Contact', icon: 'mdi-email' }
]

function switchCanvas(id: number) {
  flushBackSelectedImgs()

  // filter out click-again case
  if (id == active_canvas_id.value) { return }

  // switch the active flag for button
  active_canvas_id.value = id
  // unload the current canvas and load new canvas
  loadCanvas()
  // TODO:maybe do some caching or memory cleaning?
}

function deleteSelectImgs() {
  const selected_imgs = transformer.nodes()
  // return if no image selected
  if (selected_imgs.length === 0) { return }
  // remove all selected images
  selected_imgs.forEach((node) => { node.destroy() })
  // remove transformer from select layer
  updateTransformer()
}

function flushBackSelectedImgs() {
  const current_canvas = layers[active_canvas_id.value]
  const selected_imgs = select_layer.getChildren(
    (node) => node.getClassName() === 'Image')
  // return if no image selected
  if (selected_imgs.length === 0) { return }
  // move all selected images into current canvas
  console.log('start flushback')
  selected_imgs.forEach((node) => {
    node.moveTo(current_canvas)
  })
  // remove transformer from select layer
  updateTransformer()
}

function handleStageKeyDown(e: KeyboardEvent) {
  if (e.key === 'Delete' || e.key === 'Backspace') {
    deleteSelectImgs()
  }
}

let wheel_Y_offset = 0.0
let WheelCanvasUnlockTimeout: NodeJS.Timeout
function handleStageWheel(e: Konva.KonvaEventObject<WheelEvent>) {
  // stop default scrolling
  e.evt.preventDefault()

  if (isMovingCanvas === false) { saveTransformingPositionDiff() }
  stageChildrenDragLock()

  const pointer = stage.value?.getPointerPosition()
  if (stage.value == null || pointer == null) { return }
  const oldScale = stage.value.scaleX()

  const mousePointTo = {
    x: (pointer.x - stage.value.x()) / oldScale,
    y: (pointer.y - stage.value.y()) / oldScale,
  }

  // how to scale? Zoom in? Or zoom out?
  // const direction = e.evt.deltaY > 0 ? -1 : 1
  wheel_Y_offset -= e.evt.deltaY
  // determine zoom ratio by total wheel offset Y
  const newScale = Math.exp(wheel_Y_offset / 500)


  stage.value.scaleX(newScale)
  stage.value.scaleY(newScale)

  const newPos = {
    x: pointer.x - mousePointTo.x * newScale,
    y: pointer.y - mousePointTo.y * newScale,
  }
  stage.value.position(newPos)
  // don't unlock the canvas too quickly
  // add a debounce
  clearTimeout(WheelCanvasUnlockTimeout)
  WheelCanvasUnlockTimeout = setTimeout(() => {
    if (isTransformerDragging) { restoreTransformingPositionDiff() }
    stageChildrenDragUnLock()
  }, 20)
}

function handleStageMouseUp(e: Konva.KonvaEventObject<MouseEvent>) {
  if (e.evt.button == 1) {
    // release middle button, stop dragging canvas lock
    stageChildrenDragUnLock()
  } else {
    console.log('mouseup')
    console.log(e.target)
    // release left button, stop dragging image
    transformer.nodes().forEach((node) => {
      if (node.isDragging()) { node.stopDrag() }
    })
  }
}

let relPositionDiffs: Map<string, [number, number]> = new Map()
function saveTransformingPositionDiff() {
  const relPosition = stage.value?.getRelativePointerPosition()
  transformer.nodes().forEach((node) => {
    if (relPosition != undefined && node.className == 'Image') {
      relPositionDiffs.set(node.id(), [relPosition.x - node.x(), relPosition.y - node.y()])
    }
  })
}
function restoreTransformingPositionDiff() {
  const relPosition = stage.value?.getRelativePointerPosition()
  transformer.nodes().forEach((node) => {
    if (node.className == 'Image') {
      const relPositionDiff = relPositionDiffs.get(node.id())
      if (relPosition != undefined && relPositionDiff != undefined) {
        node.x(relPosition.x - relPositionDiff[0])
        node.y(relPosition.y - relPositionDiff[1])
      }
    }
  })
}

let isTransformerDragging = false
function stageChildrenDragLock() {
  if (isMovingCanvas === false) {
    console.log('start canvas drag')
    isTransformerDragging = transformer.isDragging()
    console.log('isdragging')
    console.log(isTransformerDragging)
    isMovingCanvas = true
    select_layer.listening(false)
    transformer.listening(false)
    transformer.stopDrag()
    transformer.stopTransform()
    layers[active_canvas_id.value].listening(false)
    select_layer.getChildren().forEach((node) => { node.stopDrag() })
  }
}

function stageChildrenDragUnLock() {
  if (isMovingCanvas === true) {
    console.log('finish canvas drag')
    layers[active_canvas_id.value].listening(true)
    transformer.listening(true)
    select_layer.listening(true)
    isMovingCanvas = false
    if (isTransformerDragging) {
      transformer.nodes().forEach((node) => {
        node.startDrag()
      })
    }
  }
}

function handleStageMouseDown(e: Konva.KonvaEventObject<MouseEvent>) {
  // enable draggable for stage for middle-mosue-click
  if (e.evt.button == 1) {
    // canvas drag
    if (stage.value?.isDragging() === false) {
      stageChildrenDragLock()
      stage.value.startDrag()
    }
    return
  }

  if (e.evt.button == 2) {
    //TODO: handle right click menu
    return
  }

  // left-mouse-click
  if (e.target.getParent()?.className === 'Transformer') {
    // if click on transformer - do nothing
    console.log('click on transformer')
  } else if (e.target === e.target.getStage()) {
    // if click on empty area - remove all transformers
    console.log('click on stage')
    flushBackSelectedImgs()
  } else if (e.target.className === 'Image') {
    // click on image
    console.log('click on image')
    console.log(e.target)

    const clicked_uuid = e.target.id()
    if (clicked_uuid == '') { return }

    if (select_layer.findOne('#' + clicked_uuid) != null) {
      // the current image is already selected, start dragging
    } else {
      // flush back the current selected image
      // TODO: add selection rectangle to multi-select 
      flushBackSelectedImgs()
      // move the image from current canvas to select layer
      e.target.moveTo(select_layer)
      updateTransformer()
    }
    if (isMovingCanvas === false && e.target.isDragging() === false) { e.target.startDrag() }
  }
}

let CanvasTouchStartTimeOut: NodeJS.Timeout
function handleStageTouchStartDebounce(e: Konva.KonvaEventObject<TouchEvent>) {
  clearTimeout(CanvasTouchStartTimeOut)
  CanvasTouchStartTimeOut = setTimeout(handleStageTouchStart, 30, e)
  handleStageTouchStart(e)
}

function handleStageTouchStart(e: Konva.KonvaEventObject<TouchEvent>) {
  // due to the debounce, it may happen after canvas moving by 2-figers gesture
  // so we need to check if the canvas is moving
  if (isMovingCanvas === true) { return }
  if (e.target == null || stage.value == null) { return }
  // only handle 1-finger touch
  if (e.evt.touches.length > 1) { return }

  if (e.target.getParent()?.className === 'Transformer') {
    // if click on transformer - do nothing
    console.log('click on transformer')
  } else if (e.target === e.target.getStage()) {
    // if click on empty area - remove all transformers
    console.log('click on stage')
    flushBackSelectedImgs()
  } else if (e.target.className === 'Image') {
    // click on image
    console.log('click on image')

    const clicked_uuid = e.target.id()
    if (clicked_uuid == '') { return }

    if ( select_layer.findOne('#' + clicked_uuid) != null) {
      // the current image is already selected, abort
    } else {
      // flush back the current selected image
      // TODO: add selection rectangle to multi-select 
      flushBackSelectedImgs()
      // move the image from current canvas to select layer
      e.target.moveTo(select_layer)
      updateTransformer()
    }
    if (isMovingCanvas === false && e.target.isDragging() === false) { e.target.startDrag() }
  }
}

function getDistance(p1: { x: number, y: number }, p2: { x: number, y: number }) {
  return Math.sqrt(Math.pow(p2.x - p1.x, 2) + Math.pow(p2.y - p1.y, 2))
}

let lastCenter: { x: number, y: number } | null = null
let lastDist = 0

function handleStageTouchMove(e: Konva.KonvaEventObject<TouchEvent>) {
  const finger_count = e.evt.touches.length
  if (finger_count < 2) { return }

  function getCenter(p1: { x: number, y: number }, p2: { x: number, y: number }) {
    return {
      x: (p1.x + p2.x) / 2,
      y: (p1.y + p2.y) / 2,
    }
  }

  e.evt.preventDefault()
  // canvas drag
  const touch1 = e.evt.touches[0]
  const touch2 = e.evt.touches[1]


  if (touch1 && touch2) {
    stageChildrenDragLock()
    // if the stage was under Konva's drag&drop
    // we need to stop it, and implement our own pan logic with two pointers
    const stage_val = stage.value
    if (stage_val == null) { return }
    if (stage_val.isDragging()) {
      stage_val.stopDrag()
    }

    const p1 = {
      x: touch1.clientX,
      y: touch1.clientY,
    }
    const p2 = {
      x: touch2.clientX,
      y: touch2.clientY,
    }

    if (!lastCenter) {
      lastCenter = getCenter(p1, p2)
      return
    }
    const newCenter = getCenter(p1, p2)

    const dist = getDistance(p1, p2)

    if (!lastDist) {
      lastDist = dist
    }

    // local coordinates of center point
    const pointTo = {
      x: (newCenter.x - stage_val.x()) / stage_val.scaleX(),
      y: (newCenter.y - stage_val.y()) / stage_val.scaleX(),
    }

    const scale = stage_val.scaleX() * (dist / lastDist)

    stage_val.scaleX(scale)
    stage_val.scaleY(scale)

    // calculate new position of the stage
    const dx = newCenter.x - lastCenter.x
    const dy = newCenter.y - lastCenter.y

    const newPos = {
      x: newCenter.x - pointTo.x * scale + dx,
      y: newCenter.y - pointTo.y * scale + dy,
    }

    stage_val.position(newPos)

    lastDist = dist
    lastCenter = newCenter
  }
}

function handleStageTouchEnd(e: Konva.KonvaEventObject<TouchEvent>) {
  // stop 'dragging' action only when all fingers are off the screen
  // for two-finger case, if one finger is still on screen, we still need to keep the dragging action
  const finger_count = e.evt.touches.length
  if (finger_count == 0) {
    // handle canvas drag stop logic
    lastCenter = null
    lastDist = 0
    stageChildrenDragUnLock()

    // handle image drag stop logic
    transformer.nodes().forEach((node) => {
      if (node.isDragging()) { node.stopDrag() }
    })
  }
}

function updateTransformer() {
  const selected_imgs = select_layer.getChildren(
    (node) => node.getClassName() === 'Image')
  if (selected_imgs.length === 0) {
    // no image selected
    transformer.remove()
    select_layer.remove()
  } else {
    // image selected 
    transformer.nodes(selected_imgs)
    if (stage.value?.getChildren().indexOf(select_layer) === -1) {
      stage.value?.add(select_layer)
    }
    select_layer.add(transformer)
  }
}


/// konva start:
function loadCanvas() {
  // add the layer to the stage
  stage.value?.removeChildren()
  stage.value?.add(layers[active_canvas_id.value])

  // TODO: maybe restore data from json
  // instantialize the ref_images into canvas

}

function unbindDragDrop() {
  const dropArea = document.getElementById('konva_container')
  dropArea?.removeEventListener('dragenter', preventDefault)
  dropArea?.removeEventListener('dragleave', preventDefault)
  dropArea?.removeEventListener('dragover', preventDefault)
  dropArea?.removeEventListener('drop', handleDropOver)
}

function bindDragDrop() {
  const dropArea = document.getElementById('konva_container')
  // TODO: change cursor to indicate if the drop is valid
  dropArea?.addEventListener('dragenter', preventDefault)
  dropArea?.addEventListener('dragleave', preventDefault)
  dropArea?.addEventListener('dragover', preventDefault)
  dropArea?.addEventListener('drop', handleDropOver)
}

function initKonva() {
  stage.value = new Konva.Stage({
    container: 'konva_container',   // id of container <div>
    width: windowWidth,
    height: windowHeight,
    draggable: false,
  })
  UpdateKonvaCanvasSize()
  stage.value?.on('mousedown', handleStageMouseDown)
  stage.value?.on('mouseup', handleStageMouseUp)
  stage.value?.on('wheel', handleStageWheel)
  stage.value?.on('touchstart', handleStageTouchStartDebounce)
  stage.value?.on('touchmove', handleStageTouchMove)
  stage.value?.on('touchend', handleStageTouchEnd)

  RefImage.fromURL('https://img.alicdn.com/tfs/TB1GzimpJTfau8jSZFwXXX1mVXa-92-126.png', (img) => { layers[0].add(img) })
  RefImage.fromURL('https://img.alicdn.com/tfs/TB1G2Wd4UT1gK0jSZFrXXcNCXXa-114-128.png', (img) => { layers[0].add(img) })
  RefImage.fromURL('https://img.alicdn.com/tfs/TB1GzimpJTfau8jSZFwXXX1mVXa-92-126.png', (img) => { layers[2].add(img) })

  // add blob konva.image
  const url = "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABgAAAAYCAYAAADgdz34AAAABHNCSVQICAgIfAhkiAAAAAlwSFlzAAAApgAAAKYB3X3/OAAAABl0RVh0U29mdHdhcmUAd3d3Lmlua3NjYXBlLm9yZ5vuPBoAAANCSURBVEiJtZZPbBtFFMZ/M7ubXdtdb1xSFyeilBapySVU8h8OoFaooFSqiihIVIpQBKci6KEg9Q6H9kovIHoCIVQJJCKE1ENFjnAgcaSGC6rEnxBwA04Tx43t2FnvDAfjkNibxgHxnWb2e/u992bee7tCa00YFsffekFY+nUzFtjW0LrvjRXrCDIAaPLlW0nHL0SsZtVoaF98mLrx3pdhOqLtYPHChahZcYYO7KvPFxvRl5XPp1sN3adWiD1ZAqD6XYK1b/dvE5IWryTt2udLFedwc1+9kLp+vbbpoDh+6TklxBeAi9TL0taeWpdmZzQDry0AcO+jQ12RyohqqoYoo8RDwJrU+qXkjWtfi8Xxt58BdQuwQs9qC/afLwCw8tnQbqYAPsgxE1S6F3EAIXux2oQFKm0ihMsOF71dHYx+f3NND68ghCu1YIoePPQN1pGRABkJ6Bus96CutRZMydTl+TvuiRW1m3n0eDl0vRPcEysqdXn+jsQPsrHMquGeXEaY4Yk4wxWcY5V/9scqOMOVUFthatyTy8QyqwZ+kDURKoMWxNKr2EeqVKcTNOajqKoBgOE28U4tdQl5p5bwCw7BWquaZSzAPlwjlithJtp3pTImSqQRrb2Z8PHGigD4RZuNX6JYj6wj7O4TFLbCO/Mn/m8R+h6rYSUb3ekokRY6f/YukArN979jcW+V/S8g0eT/N3VN3kTqWbQ428m9/8k0P/1aIhF36PccEl6EhOcAUCrXKZXXWS3XKd2vc/TRBG9O5ELC17MmWubD2nKhUKZa26Ba2+D3P+4/MNCFwg59oWVeYhkzgN/JDR8deKBoD7Y+ljEjGZ0sosXVTvbc6RHirr2reNy1OXd6pJsQ+gqjk8VWFYmHrwBzW/n+uMPFiRwHB2I7ih8ciHFxIkd/3Omk5tCDV1t+2nNu5sxxpDFNx+huNhVT3/zMDz8usXC3ddaHBj1GHj/As08fwTS7Kt1HBTmyN29vdwAw+/wbwLVOJ3uAD1wi/dUH7Qei66PfyuRj4Ik9is+hglfbkbfR3cnZm7chlUWLdwmprtCohX4HUtlOcQjLYCu+fzGJH2QRKvP3UNz8bWk1qMxjGTOMThZ3kvgLI5AzFfo379UAAAAASUVORK5CYII="
  fetch(url).then(res => res.blob()).then((blob) => {
    const img = new window.Image()
    img.src = URL.createObjectURL(blob)
    img.onload = () => {
      const RefImg = new RefImage(img)
      RefImg.image(img)
      RefImg.setSize({ width: 100, height: 100 })
      layers[0].add(RefImg)
    }
  })

  loadCanvas()
  // by default Konva prevent some events when node is dragging
  // it improve the performance and work well for 95% of cases
  // we need to enable all events on Konva, even when we are dragging a node
  // so it triggers touchmove correctly
  Konva.hitOnDragEnabled = true
  Konva.dragButtons = [0]
  // Konva.pixelRatio = 1
}
</script>


<template>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, user-scalable=no">
  <v-layout class='rounded rounded-md'>
    <v-app-bar fluid flat density='compact' color='#efefef' id='toolbar'>
      <v-btn variant='flat' color='primary' class='ml-3'>
        Menu
        <v-menu activator='parent'>
          <v-list nav density='compact'>
            <v-list-item v-for='item in menu_items'>
              <v-list-item-title> {{ item.title }} </v-list-item-title>
            </v-list-item>
          </v-list>
        </v-menu>
      </v-btn>

      <v-btn-group rounded='0' class='ml-3' density='compact'>
        <v-btn :variant="active_canvas_id === i ? 'tonal' : 'text'" class='navbtn px-1 ma-1' v-for='i in Array(6).keys()'
          @click='switchCanvas(i)'>
          {{ i }}
        </v-btn>
      </v-btn-group>
    </v-app-bar>
    <v-main fluid id="konva_container">
    </v-main>
  </v-layout>
</template>

<style scoped>
header {
  position: absolute;
  line-height: 1.5;
  max-height: 100vh;
}

.v-toolbar {
  z-index: 100;
}


.navbtn {
  min-width: 2em;
}
</style>