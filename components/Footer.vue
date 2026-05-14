<template>
  <div class="footer-brand">
    <img 
      v-if="showLogo" 
      src="/resizes-logo.webp" 
      alt="Resizes" 
      class="footer-logo"
      @error="showLogo = false"
    />
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const showLogo = ref(true)
</script>

<style scoped>
.footer-brand {
  position: fixed;
  bottom: 16px;
  right: 20px;
  z-index: 100;
  opacity: 0.6;
  transition: opacity 0.2s;
}

.footer-brand:hover {
  opacity: 0.9;
}

.footer-logo {
  height: 16px;
  width: auto;
  max-width: 80px;
  image-rendering: pixelated;
  image-rendering: -moz-crisp-edges;
  image-rendering: crisp-edges;
}
</style>

