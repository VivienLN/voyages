<script setup>
  import Earth3d from '@/components/Earth3d.vue'
  import destinations from '@/data/destinations.json'
  import { reactive, ref } from 'vue'

  const state = reactive({
    current: {}
  })
  const index = ref(0)

  updateCurrent()

  function previous() {
    let target = index.value - 1
    index.value = target < 0 ? destinations.length - 1 : target
    updateCurrent()
  }

  function next() {
    let target = index.value + 1
    index.value = target > destinations.length - 1 ? 0 : target
    updateCurrent()
  }

  function updateCurrent() {
    state.current = destinations[index.value]
  }
</script>

<template>
  <main>
    {{ destinations.length }}
    <Earth3d v-bind="state.current" />
    <button @click="previous">Précédent</button>
    <button @click="next">Suivant</button>
  </main>
</template>
