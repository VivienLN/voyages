<script setup>
  import Earth3d from '@/components/Earth3d.vue'
  import destinations from '@/data/destinations.json'
  import { reactive, ref } from 'vue'

  const state = reactive({
    current: {},
    previous: {},
    next: {},
  })
  const index = ref(0)

  updateCurrent()

  function previous() {
    index.value = getPreviousIndex()
    updateCurrent()
  }

  function next() {
    index.value = getNextIndex()
    updateCurrent()
  }

  function getPreviousIndex() {
    let target = index.value - 1
    return target < 0 ? destinations.length - 1 : target
  }

  function getNextIndex() {
    let target = index.value + 1
    return target > destinations.length - 1 ? 0 : target
  }

  function updateCurrent() {
    state.current = destinations[index.value]
    state.previous = destinations[getPreviousIndex()]
    state.next = destinations[getNextIndex()]
  }
</script>

<template>
  <main>
    {{ destinations.length }}
    <Earth3d 
      :previous="state.previous"
      :next="state.next"
      v-bind="state.current"
      @clickPrevious="previous"
      @clickNext="next"
    />
    <button @click="previous">Précédent</button>
    <button @click="next">Suivant</button>
  </main>
</template>
