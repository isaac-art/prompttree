<script setup>
import { ref } from 'vue'
import { Handle, Position } from '@vue-flow/core'

const onConnect = (params) => console.log('handle onConnect', params)

const props = defineProps({
  data: {
    type: Object,
    required: true,
  },
})

</script>

<template>
  <!-- <Handle type="target" :position="Position.Top" /> -->
  <div class="inputprompt">
    
    <p>{{ props.data.prompt }}</p>
    <button @click="submit">submit</button>

    <Handle type="source" :position="Position.Bottom" :on-connect="onConnect" />
  </div>
</template>

<style scoped>
.inputprompt{
  border: 1px solid #111;
  padding-top: 0px;
  padding-left: 10px;
  padding-right: 10px;
  padding-bottom: 10px;
  border-radius: 4px;
  max-width: 600px;
}
</style>
