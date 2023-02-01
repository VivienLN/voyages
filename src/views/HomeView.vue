<script setup>
  import Earth3d from '@/components/Earth3d.vue'
  import { reactive, ref } from 'vue'

  const state = reactive({
    title: "Hello world 0",
    count: 0,
  })

  function increment() {
    state.count++
    state.title = "Hello world " + state.count
    console.log("increment", state)
  }
</script>

<template>
  <main>
    Hello world
    <button @click="increment">Change title</button>
    <Earth3d v-bind="state" />
  </main>
</template>
