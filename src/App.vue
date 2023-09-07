<script setup lang="ts">
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
</script>



<template>
  <header>
    <v-toolbar absolute density="compact">
      <v-btn variant="flat" color="primary">
        Menu
        <v-menu activator="parent">
          <v-list nav density="compact">
            <v-list-item v-for="(item, index) in items" :key="index" :value="index">
              <v-list-item-title> {{ item.title }} </v-list-item-title>
            </v-list-item>
          </v-list>
        </v-menu>
      </v-btn>

      <v-btn-group rounded="0" class="ml-3" density="compact">
        <v-btn :variant="item.active ? 'tonal' : 'text'" class="navbtn px-1 ma-1" v-for="(item, index) in canvases"
          :key="index" @click="switchCanvas(item.id)">
          {{ item.id }}
        </v-btn>
      </v-btn-group>

      <v-toolbar-items class="ml-6">
        <ul>
          <li>Screen Width: {{ windowWidth }}</li>
          <li>Screen Height: {{ windowHeight }}</li>
        </ul>
      </v-toolbar-items>
    </v-toolbar>
  </header>

  <!-- TODO: 两个layer，一个是编辑中的对象（可能多个），另一个是剩余的 -->
  <!-- TODO: v-stage最终也会作为动态对象保存，保存一个-->
  <v-stage :width="windowWidth" :height="windowHeight">
    <v-layer>
      <v-circle :config="configCircle"></v-circle>
    </v-layer>
    <v-layer>
      <v-circle :config="configCircle"></v-circle>
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

<script lang="ts">
const canvases = ref([
  { id: 1, title: "Canvas 1", active: true },
  { id: 2, title: "Canvas 2", active: false },
  { id: 3, title: "Canvas 3", active: false },
  { id: 4, title: "Canvas 4", active: false },
  { id: 5, title: "Canvas 5", active: false },
  { id: 6, title: "Canvas 6", active: false },
])
export default {
  data: () => ({
    configKonva: {
      width: 100,
      height: 100
    },
    configCircle: {
      x: 100,
      y: 100,
      radius: 70,
      fill: "red",
      stroke: "black",
      strokeWidth: 4,
      draggable: "true"
    },
    items: [
      { title: "Home", icon: "mdi-home" },
      { title: "About", icon: "mdi-help-box" },
      { title: "Contact", icon: "mdi-email" }
    ],
    canvases
  }),
};

function switchCanvas(id: number) {
  // TODO: filter out self-clicking cases
  // switch the active flag for button
  canvases.value.forEach((item) => {
    if (item.id === id) {
      item.active = true;
    } else {
      item.active = false;
    }
  });
  // TODO: unload the current canvas and load new canvas
}
</script>

