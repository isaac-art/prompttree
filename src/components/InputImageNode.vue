<script setup>
import { ref } from 'vue'
import { Handle, Position } from '@vue-flow/core'

defineProps({
  msg: String,
})

const count = ref(0)
</script>

<template>
  <h1>{{ msg }}</h1>
</template>

<style scoped>

</style>
