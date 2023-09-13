<script setup lang='ts'>
import { ref, onMounted, onUnmounted } from 'vue'
import Konva from 'konva';
import { useTheme } from 'vuetify/lib/framework.mjs';


let windowWidth: number = window.innerWidth
let windowHeight: number = window.innerHeight / 2

let CanvasResizeTimeOut: NodeJS.Timeout
const UpdateKonvaCanvasSize = () => {
  const toolbar:HTMLDivElement | null = document.querySelector('#toolbar');
  if (toolbar == null) { return }
  windowWidth =  window.innerWidth
  windowHeight = window.innerHeight- toolbar.offsetHeight
  // update Konva Canavs size
  stage.value?.width(windowWidth)
  stage.value?.height(windowHeight)
}
const handleWindowResize = () => {
  clearTimeout(CanvasResizeTimeOut)

  CanvasResizeTimeOut = setTimeout(UpdateKonvaCanvasSize, 300)
}

onMounted(() => {
  initKonva()
  window.addEventListener('resize', handleWindowResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleWindowResize)
})


const transformerConfig = {
  anchorStroke: 'white',
  anchorFill: '#168e99',
  anchorSize: 10,
  anchorCornerRadius: 5,
  borderStroke: '#168e99',
  borderStrokeWidth: 2,
  borderDash: [1, 0],
  keepratio: true,
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
    this.id(self.crypto.randomUUID())
  }
}

class RefLayers extends Konva.Layer {
  index: number
  constructor(id: number) {
    super()
    this.index = id
    this.addName('refLayer')
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

function flushBackSelectedImgs() {
  const selected_imgs = select_layer.getChildren(
    (node) => node.getClassName() === 'Image')
  // return if no image selected
  if (selected_imgs.length === 0) { return }
  // move all selected images into current canvas
  selected_imgs.forEach((node) => {
    node.moveTo(layers[active_canvas_id.value])
  })
  // remove transformer from select layer
  updateTransformer()
}

function handleStageMouseDown(e: any) {
  const current_canvas = layers[active_canvas_id.value]
  if (e.target == null || stage.value == null) { return }
  else if (e.target === e.target.getStage()) {
    // if click on empty area - remove all transformers
    console.log('click on stage')
    flushBackSelectedImgs()
    // enable draggable for stage
    stage.value.setDraggable(true)
  } else if (e.target.getParent().className === 'Transformer') {
    // if click on transformer - do nothing
    console.log('click on transformer')
    // do nothing
  } else {
    // click on image
    console.log('click on image')

    let clicked_uuid = e.target.id()
    // check uuid is not null and stage exists
    if (clicked_uuid == '') { return }

    const refimg: RefImage = current_canvas.findOne('#' + clicked_uuid)
    if (refimg == null) {
      // the current image is already selected, abort
      return
    } else {
      // disable draggable for stage
      stage.value.setDraggable(false)
      // move the image from current canvas to select layer
      refimg.moveTo(select_layer)

      updateTransformer()
    }

  }
}

function updateTransformer() {
  const selected_imgs = select_layer.getChildren(
    (node) => node.getClassName() === 'Image')
  if (selected_imgs.length === 0) {
    // no image selected
    transformer.remove()
  } else {
    // image selected 
    transformer.nodes(selected_imgs)
    select_layer.add(transformer)
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

  // way to add blob konva.image
  // const url = "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg=="
  // fetch(url).then(res=>res.blob()).then((blob)=>{
  //   const img = new Image()
  //   img.src = URL.createObjectURL(blob)
  //   img.onload = () => {
  //     const pattern = new Konva.Image({
  //       x: 100,
  //       y: 100,
  //       image: img,
  //       width: 1000,
  //       height: 1000,
  //       draggable: true,
  //     })
  //     layer.add(pattern)
  //     layer.batchDraw()
  //   }
  // })
}

function initKonva() {
  stage.value = new Konva.Stage({
    container: 'konva_container',   // id of container <div>
    width: windowWidth,
    height: windowHeight
  })
  UpdateKonvaCanvasSize()
  stage.value?.on('mousedown', handleStageMouseDown)

  const img0 = new RefImage('https://konvajs.github.io/assets/lion.png')
  const img1 = new RefImage('https://konvajs.github.io/assets/lion.png')
  const img2 = new RefImage('https://konvajs.github.io/assets/lion.png')
  layers[0].add(img0)
  layers[0].add(img1)
  layers[2].add(img2)

  loadCanvas()
}


</script>



<template>
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

  <!-- TODO: 两个layer，一个是编辑中的对象（可能多个），另一个是剩余的 -->
  <!-- TODO: v-stage最终也会作为动态对象保存，保存一个-->
  <!-- <v-stage ref='stage' :width='windowWidth' :height='windowHeight' @mousedown="handleStageMouseDown"
    @touchStart='handleStageMouseDown'>
    <v-layer ref='layer_select'>
      <v-image v-for='selected_img in selected_imgs' :config='selected_img' />
    </v-layer>
    <v-layer>
      <v-image v-for='ref_image in canvases[active_canvas_id].images' :config='ref_image' />
    </v-layer>
  </v-stage> -->
</template>

<style scoped>
header {
  position:absolute;
  line-height: 1.5;
  max-height: 100vh
}

.v-toolbar {
  z-index: 100
}

.navbtn {
  min-width: 2em
}
</style>