<script setup>
import { ref } from 'vue'
import { getNodesInside, Handle, Position } from '@vue-flow/core'
import { NodeToolbar } from '@vue-flow/node-toolbar'
import axios from 'axios'


const onConnect = (params) => console.log('handle onConnect', params)

const props = defineProps({
  data: {
    type: Object,
    required: true,
  },
  selected: {
    type: Boolean,
    required: true,
  },
  position: {
    type: Object,
    required: true,
  },
  id: {
    type: String,
    required: true,
  },
  events : {
    type: Object,
    required: true,
  },
  type : {
    type: String,
    required: true,
  },
  resizing : {
    type: Boolean,
    required: true,
  },
  dragging : {
    type: Boolean,
    required: true,
  },
  connectable : {
    type: Boolean,
    required: true,
  },
  dimensions : {
    type: Object,
    required: true,
  },
  isValidTargetPos : {
    required: false,
  },
  isValidSourcePos : {
    required: false,
  },
  parent : {
    required: false,
  },
  zIndex : {
    type: Number,
    required: true,
  },
  targetPosition : {
    type: String,
  },
  sourcePosition : {
    type: String,
  },
  label : {
    required: true,
  },
  dragHandle : {
    required: true,
  },
})

const emit = defineEmits(['updateNodeInternals','forkPrompt', 'unselect', 'generate', 'deleteNode'])

function forkPrompt(){
  // console.log('forkPrompt')
  emit('forkPrompt')
}

function onPromptChange(e){
  if (e.key == 'Enter'){
    console.log('onPromptChange', e.target.value)
    gen()
  }
}

function gen(){
  emit('unselect')
  emit('generate')
}

function deleteNode(){
  emit('deleteNode')
}

</script>

<template>  
  <div id="delete" v-if="props.selected"> 
    <button @click="deleteNode">X</button> 
  </div>
  <div class="promptnodecontainer">
    <Handle type="target" :position="Position.Left"  :on-connect="onConnect"/>
    <Handle type="source" :position="Position.Right" :on-connect="onConnect"/>

    <NodeToolbar
      style="display: flex; gap: 0.5rem; align-items: center"
      :is-visible="props.selected"
      :position="Position.Bottom"
    >
    <span v-if="!props.data.generating">
      <button @click="gen"><span v-if="props.data.images.length ==0">gen</span><span v-else>regen</span></button>
      <button @click="forkPrompt" v-if="props.data.images.length > 0">fork</button>
    </span>
    </NodeToolbar>

    <div class="promptnode" :class="{generating: props.data.generating, selected: props.selected}">

      <textarea type="text" 
        v-if="props.selected && props.data.images.length == 0 && !props.data.generating" 
        :focus="props.selected"
        v-model="props.data.prompt"
        @keypress="onPromptChange"
      ></textarea>
      <p v-else>
        <span v-if="props.data.generating">...generating...</span>
        <span v-else>
          {{ props.data.prompt }}
        </span>
      </p>
      <span v-if="selected && props.data.images.length == 0 && !props.data.generating">
      <hr>
        <input class="size_input" type="number" v-model="props.data.width" min="128" max="768" step="64" style="width: auto;"/>
        x
        <input class="size_input" type="number" v-model="props.data.height" min="128" max="768" step="64" style="width: auto;"/>
      
      </span>
      <hr>
      <span v-for="(image, idx) in props.data.images"
          class="image" >
          <img :src="image.url"/>
      </span>

      <!-- <button v-if="props.data.images.length == 0" @click="enhance">enhance</button>
      <button v-if="props.data.images.length == 0" @click="submit">generate</button> -->

    </div>
  </div>
</template>

<style scoped>
#delete{
  position: relative;
  float: right;
  top: -16px;
  right: -16px;
}
#delete button{
  background-color: rgb(91, 25, 25);
  width: 36px;
  height: 36px;
  border-radius: 18px;
  font-size: larger;
}
#delete button:hover{
  background-color: firebrick;
}
.promptnode{
  border: 2px solid #555;
  padding-top: 0px;
  padding-left: 10px;
  padding-right: 10px;
  padding-bottom: 10px;
  border-radius: 4px;
  max-width: 256px;
  width: 256px;
  height: auto;
  min-height: 160px;
  background: #fff;
}
.generating{
  border: 2px dashed rgb(231, 151, 13) !important;
  border-radius: 4px;
}
.selected {
  border: 2px dashed rgb(179, 14, 72) !important;
  border-radius: 4px;
}
textarea{
  margin-top: -17px;
  margin-bottom: 16px;
  padding:0px;
  width: 100%;
  min-height: 160px;
  font-size: 1rem;
  font-family: 'IBM Plex Mono', monospace;
  border: none;
  background-color: transparent;
  /* color: #fff; */
}
img{
  width: 64px;
  height: auto;
}
</style>
