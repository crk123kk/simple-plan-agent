<script setup lang="ts">
import { ref } from 'vue'

const response = ref('')

const handlePing = async () => {
  response.value = await window.api.ping()
}
</script>

<template>
  <main class="app-shell">
    <section class="card">
      <p class="eyebrow">Electron + Vue starter</p>
      <h1>It works.</h1>
      <p class="description">A minimal desktop app with Vue in the renderer and a safe preload bridge.</p>
      <button type="button" @click="handlePing">Ping main process</button>
      <p v-if="response" class="result">Response: {{ response }}</p>
    </section>
  </main>
</template>