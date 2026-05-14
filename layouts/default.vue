<template>
  <div class="slidev-layout default" :class="layoutClass">
    <slot />
    <Footer />
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import Footer from '../components/Footer.vue'

const props = defineProps<{
  class?: string
}>()

const layoutClass = computed(() => props.class)
</script>
