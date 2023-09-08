<script setup lang='ts'>
import { Canvas } from 'konva/lib/Canvas';
import { Image } from 'konva/lib/shapes/Image';
import { Stage } from 'konva/lib/Stage';
import { Transformer } from 'konva/lib/shapes/Transformer';
import { setgroups } from 'process';
import { arrayBuffer } from 'stream/consumers';
import { ref, onMounted, onUnmounted } from 'vue'

const windowWidth = ref(window.innerWidth)
const windowHeight = ref(window.innerHeight)

const handleWindowResize = () => {
  windowWidth.value = window.innerWidth;
  windowHeight.value = window.innerHeight
}

onMounted(() => {
  window.addEventListener('resize', handleWindowResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleWindowResize)
})


const transformer = ref(null)
const stage = ref(null)

class RefImage {
  name: string
  image: HTMLImageElement
  x: number
  y: number
  rotation: number
  scaleX: number
  scaleY: number
  width: number
  height: number
  readonly draggable = true

  constructor(src: string) {
    this.name = self.crypto.randomUUID()
    this.image = new window.Image()
    this.image.src = src
    this.x = 0
    this.y = 0
    this.width = 200
    this.height = 300
    this.scaleX = 1.0
    this.scaleY = 1.0
    this.rotation = 0
  }
}

class RefCavnas {
  id: number
  active: boolean
  images: RefImage[]
  constructor(id: number, active: boolean) {
    this.id = id
    this.images = []
    this.active = active
  }
}

const active_canvas_id = ref(0)
const canvases = Array.from(Array(6), (v, k) => (new RefCavnas(k, v == 0)))
var selectedImgUUID: string = ''

canvases[0].images.push(new RefImage('https://konvajs.github.io/assets/lion.png'))

const configTransFormer = ref({
  anchorStroke: 'red',
  anchorFill: 'yellow',
  anchorSize: 20,
  borderStroke: 'green',
  borderDash: [3, 3],
  keepratio: true,
})

const menu_items = [
  { title: 'Home', icon: 'mdi-home' },
  { title: 'About', icon: 'mdi-help-box' },
  { title: 'Contact', icon: 'mdi-email' }
]

function switchCanvas(id: number) {
  // filter out click-again case
  if (id == active_canvas_id.value) { return }

  // switch the active flag for button
  active_canvas_id.value = id
  // TODO: unload the current canvas and load new canvas
  //       maybe do some caching or memory cleaning?
  // TODO: clean up the transformer
}

function handleStageMouseDown(e) {
  console.log(e)
  console.log("stage")
  console.log(stage.value)
  // if click on empty area - remove all transformers
  if (e.target === stage.value) {
    transformer.detach()
    stage.value?.batchDraw()
  }
}

function handleImageMouseDown(e) {
  console.log(stage.value)
  stage.value?.find('Transformer').forEach((t) => t.destroy());
  // create new transformer
  transformer.value?.attachTo(e.target) // no idea what to do here ...
  stage.value?.batchDraw()
}

</script>



<template>
  <header>
    <meta name='viewport' content='width=device-width, initial-scale=1.0, user-scalable=no'>

    <v-toolbar absolute density='compact'>
      <v-btn variant='flat' color='primary'>
        Menu
        <v-menu activator='parent'>
          <v-list nav density='compact'>
            <v-list-item v-for='(item, index) in menu_items' :key='index' :value='index'>
              <v-list-item-title> {{ item.title }} </v-list-item-title>
            </v-list-item>
          </v-list>
        </v-menu>
      </v-btn>

      <v-btn-group rounded='0' class='ml-3' density='compact'>
        <v-btn :variant="active_canvas_id === item.id ? 'tonal' : 'text'" class='navbtn px-1 ma-1'
          v-for='(item, index) in canvases' :key='index' @click='switchCanvas(item.id)'>
          {{ item.id }}
        </v-btn>
      </v-btn-group>

      <v-toolbar-items class='ml-6'>
        <ul>
          <li>Screen Width: {{ windowWidth }}</li>
          <li>Screen Height: {{ windowHeight }}</li>
        </ul>
      </v-toolbar-items>
    </v-toolbar>
  </header>

  <!-- TODO: 两个layer，一个是编辑中的对象（可能多个），另一个是剩余的 -->
  <!-- TODO: v-stage最终也会作为动态对象保存，保存一个-->
  <v-stage :width='windowWidth' :height='windowHeight' :ref="stage" @mousedown="handleStageMouseDown"
    @touchStart="handleStageMouseDown">
    <v-layer>
      <v-image v-for='ref_image in canvases[active_canvas_id].images' :config='ref_image'
        @mousedown="handleImageMouseDown" @touchStart="handleImageMouseDown" />
      <!-- v-transformer不靠谱，考虑用vanilla json做一个 -->
      <v-transformer :config='configTransFormer' :ref='transformer' />
    </v-layer>
  </v-stage>
</template>

<style scoped>
header {
  line-height: 1.5;
  max-height: 100vh;
}

.v-toolbar {
  z-index: 1000;
}

.navbtn {
  min-width: 2em
}
</style>