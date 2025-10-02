<script setup lang="ts">
import { ref } from 'vue'

// DATA - MODEL
const counter = ref(10)

// METHODS - CONTROLLER
const increment = () => {
  counter.value++
}
const reset = () => {
  counter.value = 0
}
</script>

<template>
  <h1>Hello World</h1>
  <button @click="increment">Increment</button>
  <button @click="reset">Reset</button>
  <p style="color: black; font-size: 50px">{{ counter }}</p>
</template>
