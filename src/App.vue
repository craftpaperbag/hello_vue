<script setup>
import { ref } from 'vue'

const message = ref('Hello Vue')

const count = ref(0)

function countup(){
  count.value++
}
</script>

<template>
  <h1>{{ message }}</h1>
  <button @click="countup">count is {{ count }}</button>

</template>

<style scoped>
  
</style>
