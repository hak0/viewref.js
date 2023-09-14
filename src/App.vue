<script setup lang='ts'>
import { ref, onMounted, onUnmounted } from 'vue'
import Konva from 'konva';
import { nanoid } from 'nanoid'


let windowWidth: number = window.innerWidth
let windowHeight: number = window.innerHeight / 2

let CanvasResizeTimeOut: NodeJS.Timeout
const UpdateKonvaCanvasSize = () => {
  const toolbar: HTMLDivElement | null = document.querySelector('#toolbar');
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
    const blob = item.getAsFile();
    if (blob == null) { return }
    const reader = new FileReader();
    // create a new RefImage into the current canvas
    reader.onload = () => {
      const RefImg = new RefImage(reader.result as string)
      layers[active_canvas_id.value].add(RefImg)
    };
    reader.readAsDataURL(blob);
  }
}

const preventDefault = (e: any) => { e.preventDefault() }

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
  constructor(src: string) {
    const data = new window.Image()
    data.src = src
    super({
      draggable: true,
      image: data
    })
    this.id(nanoid())
  }
}

class RefLayers extends Konva.Layer {
  index: number
  constructor(id: number) {
    super()
    this.index = id
    this.addName('refLayer')
    this.draggable(false)
  }
}

const active_canvas_id = ref(0)
const layers = Array.from(Array(6), (v, k) => (new RefLayers(k)))
const select_layer = new Konva.Layer()
// konva elements
const stage = ref<Konva.Stage | null>(null)

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
  const selected_imgs = select_layer.getChildren(
    (node) => node.getClassName() === 'Image')
  // return if no image selected
  if (selected_imgs.length === 0) { return }
  // remove all selected images
  selected_imgs.forEach((node) => { node.remove() })
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
  selected_imgs.forEach((node) => {
    node.moveTo(current_canvas)
  })
  // remove transformer from select layer
  updateTransformer()
  current_canvas.listening(true)
}

function handleStageKeyDown(e: KeyboardEvent) {
  if (e.key === 'Delete' || e.key === 'Backspace') {
    deleteSelectImgs()
  }
}

function handleStageWheel(e: any) {
  // stop default scrolling
  e.evt.preventDefault();

  // zoom ratio
  const scaleBy = 1.1

  var pointer = stage.value?.getPointerPosition();
  if (stage.value == null || pointer == null) { return }
  var oldScale = stage.value.scaleX();

  var mousePointTo = {
    x: (pointer.x - stage.value.x()) / oldScale,
    y: (pointer.y - stage.value.y()) / oldScale,
  };


  // how to scale? Zoom in? Or zoom out?
  let direction = e.evt.deltaY > 0 ? 1 : -1;

  // when we zoom on trackpad, e.evt.ctrlKey is true
  // in that case lets revert direction
  if (e.evt.ctrlKey) {
    direction = -direction;
  }

  var newScale = direction > 0 ? oldScale * scaleBy : oldScale / scaleBy;

  stage.value.scale({ x: newScale, y: newScale });

  var newPos = {
    x: pointer.x - mousePointTo.x * newScale,
    y: pointer.y - mousePointTo.y * newScale,
  };
  stage.value.position(newPos);
}

function handleStageMouseDown(e: any) {
  const current_canvas = layers[active_canvas_id.value]
  if (e.target == null || stage.value == null) { return }

  // enable draggable for stage for middle-mosue-click
  if (e.evt.type === 'mousedown' && e.evt.button == 1) {
    console.log('canvas drag')
    e.target.stopDrag()
    stage.value.startDrag()
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
  } else {
    // click on image
    console.log('click on image')

    let clicked_uuid = e.target.id()
    if (clicked_uuid == '') { return }

    const refimg: RefImage = current_canvas.findOne('#' + clicked_uuid)
    if (refimg == null) {
      // the current image is already selected, start dragging
      return
    } else {
      // flush back the current selected image
      // TODO: add selection rectangle to multi-select 
      flushBackSelectedImgs()
      // move the image from current canvas to select layer
      refimg.moveTo(select_layer)
      updateTransformer()
      refimg.startDrag()
    }
  }
}

let CanvasTouchStartTimetOut: NodeJS.Timeout
function handleStageTouchStartDebounce(e : any) {
  clearTimeout(CanvasTouchStartTimetOut)
  CanvasTouchStartTimetOut = setTimeout(handleStageTouchStart, 100, e)
}
function handleStageTouchStart(e: any) {
  const current_canvas = layers[active_canvas_id.value]
  console.log(e)
  console.log(e.evt.type)

  if (e.target == null || stage.value == null) { return }
  // enable draggable for stage for more-than-one finger touch
  if (e.evt.type === 'touchstart' && e.evt.touches.length > 1) {
    console.log('canvas drag')
    e.target.stopDrag()
    stage.value.startDrag()
    return
  }

  // one-finter touch
  if (e.target.getParent()?.className === 'Transformer') {
    // if click on transformer - do nothing
    console.log('click on transformer')
  } else if (e.target === e.target.getStage()) {
    // if click on empty area - remove all transformers
    console.log('click on stage')
    flushBackSelectedImgs()
  } else {
    // click on image
    console.log('click on image')

    let clicked_uuid = e.target.id()
    if (clicked_uuid == '') { return }

    const refimg: RefImage = current_canvas.findOne('#' + clicked_uuid)
    if (refimg == null) {
      // the current image is already selected, abort
      e.target.startDrag()
      return
    } else {
      // flush back the current selected image
      // TODO: add selection rectangle to multi-select 
      flushBackSelectedImgs()
      // move the image from current canvas to select layer
      refimg.moveTo(select_layer)
      current_canvas.listening(false)
      updateTransformer()
      refimg.startDrag()
    }

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
    select_layer.add(transformer)
    if (stage.value?.getChildren().indexOf(select_layer) === -1) {
      stage.value?.add(select_layer)
    }
    select_layer.batchDraw()
  }
}


/// konva start:
function loadCanvas() {
  // add the layer to the stage
  stage.value?.removeChildren()
  stage.value?.add(layers[active_canvas_id.value])
  stage.value?.add(select_layer)

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
    transformerEnabled: 'position'
  })
  UpdateKonvaCanvasSize()
  stage.value?.on('mousedown', handleStageMouseDown)
  stage.value?.on('wheel', handleStageWheel)
  stage.value?.on('touchstart', handleStageTouchStartDebounce)

  const img0 = new RefImage('https://www.baidu.com/img/PCfb_5bf082d29588c07f842ccde3f97243ea.png')
  const img1 = new RefImage('https://img.alicdn.com/tfs/TB1G2Wd4UT1gK0jSZFrXXcNCXXa-114-128.png')
  const img2 = new RefImage('https://img.alicdn.com/tfs/TB1GzimpJTfau8jSZFwXXX1mVXa-92-126.png')
  layers[0].add(img0)
  layers[0].add(img1)
  layers[2].add(img2)

  // add blob konva.image
  const url = "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABgAAAAYCAYAAADgdz34AAAABHNCSVQICAgIfAhkiAAAAAlwSFlzAAAApgAAAKYB3X3/OAAAABl0RVh0U29mdHdhcmUAd3d3Lmlua3NjYXBlLm9yZ5vuPBoAAANCSURBVEiJtZZPbBtFFMZ/M7ubXdtdb1xSFyeilBapySVU8h8OoFaooFSqiihIVIpQBKci6KEg9Q6H9kovIHoCIVQJJCKE1ENFjnAgcaSGC6rEnxBwA04Tx43t2FnvDAfjkNibxgHxnWb2e/u992bee7tCa00YFsffekFY+nUzFtjW0LrvjRXrCDIAaPLlW0nHL0SsZtVoaF98mLrx3pdhOqLtYPHChahZcYYO7KvPFxvRl5XPp1sN3adWiD1ZAqD6XYK1b/dvE5IWryTt2udLFedwc1+9kLp+vbbpoDh+6TklxBeAi9TL0taeWpdmZzQDry0AcO+jQ12RyohqqoYoo8RDwJrU+qXkjWtfi8Xxt58BdQuwQs9qC/afLwCw8tnQbqYAPsgxE1S6F3EAIXux2oQFKm0ihMsOF71dHYx+f3NND68ghCu1YIoePPQN1pGRABkJ6Bus96CutRZMydTl+TvuiRW1m3n0eDl0vRPcEysqdXn+jsQPsrHMquGeXEaY4Yk4wxWcY5V/9scqOMOVUFthatyTy8QyqwZ+kDURKoMWxNKr2EeqVKcTNOajqKoBgOE28U4tdQl5p5bwCw7BWquaZSzAPlwjlithJtp3pTImSqQRrb2Z8PHGigD4RZuNX6JYj6wj7O4TFLbCO/Mn/m8R+h6rYSUb3ekokRY6f/YukArN979jcW+V/S8g0eT/N3VN3kTqWbQ428m9/8k0P/1aIhF36PccEl6EhOcAUCrXKZXXWS3XKd2vc/TRBG9O5ELC17MmWubD2nKhUKZa26Ba2+D3P+4/MNCFwg59oWVeYhkzgN/JDR8deKBoD7Y+ljEjGZ0sosXVTvbc6RHirr2reNy1OXd6pJsQ+gqjk8VWFYmHrwBzW/n+uMPFiRwHB2I7ih8ciHFxIkd/3Omk5tCDV1t+2nNu5sxxpDFNx+huNhVT3/zMDz8usXC3ddaHBj1GHj/As08fwTS7Kt1HBTmyN29vdwAw+/wbwLVOJ3uAD1wi/dUH7Qei66PfyuRj4Ik9is+hglfbkbfR3cnZm7chlUWLdwmprtCohX4HUtlOcQjLYCu+fzGJH2QRKvP3UNz8bWk1qMxjGTOMThZ3kvgLI5AzFfo379UAAAAASUVORK5CYII="
  fetch(url).then(res => res.blob()).then((blob) => {
    const img = new Image()
    img.src = URL.createObjectURL(blob)
    img.onload = () => {
      const RefImg = new RefImage('')
      RefImg.image(img)
      RefImg.setSize({ width: 100, height: 100 })
      layers[0].add(RefImg)
    }
  })

  loadCanvas()
}
</script>


<template>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
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