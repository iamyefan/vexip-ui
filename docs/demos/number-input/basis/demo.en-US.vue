<template>
  <NumberInput v-model:value="value" style="max-width: 300px;"></NumberInput>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const value = ref<number | null>(null)
</script>
