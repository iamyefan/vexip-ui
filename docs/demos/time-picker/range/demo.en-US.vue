<template>
  <TimePicker v-model:value="time" is-range style="max-width: 300px;"></TimePicker>
  <p>
    Current Time Range: {{ time }}
  </p>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const time = ref<string[]>([])

    return { time }
  }
})
</script>
