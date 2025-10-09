<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb'

// DATA - MODEL
const storage = ref()
const counter = ref(10)

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
});

const initDatabase = () => {
  console.log('=> Connexion à la base de donnes');
  const db = new PouchDB('http://elia:admin@localhost:5984/')
  if (db) {
    console.log("Connected to collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}
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
  <p>heyyy</p>
</template>
