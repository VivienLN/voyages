<script setup>
  import Earth3d from '@/components/Earth3d.vue'
  import destinations from '@/data/destinations.json'
  import { onMounted, onUnmounted, reactive, ref } from 'vue'

  const state = reactive({
    current: {},
    previous: {},
    next: {},
  })
  const index = ref(0)

  // First update
  updateCurrent()

  // Mouse wheel event with throttle
  const WHEEL_THROTTLE = 800
  var wheelThrottleTimeout = null
  var wheelThrottle = false

  function onMousewheel(event) {
    if (wheelThrottle) {
      return
    }

    wheelThrottle = true

    event.deltaY > 0 ? next() : previous()

    wheelThrottleTimeout = setTimeout(() => {
      wheelThrottle = false
    }, WHEEL_THROTTLE)
  }

  // Mounted / unmounted events
  onMounted(() => {
    // Nothing
  })
  onUnmounted(() => {
    clearTimeout(wheelThrottleTimeout)
  })

  // Navigation functions
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
      @wheel.passive="onMousewheel"
      @clickPrevious="previous"
      @clickNext="next"
    />
    <button @click="previous">Précédent</button>
    <button @click="next">Suivant</button>
  </main>
</template>
