<script setup>

import { defineComponent, ref, onMounted, reactive, watch } from "vue";
import { Panel, PanelPosition, VueFlow, isNode, useVueFlow , Position, MarkerType} from '@vue-flow/core'

import { Background } from '@vue-flow/background'
import { Controls } from '@vue-flow/controls'

// import local components from components folder
import OutputImageNode from './components/OutputImageNode.vue'
import InputImageNode from './components/InputImageNode.vue'
import InputPromptNode from './components/InputPromptNode.vue'
import PromptNode from "./components/PromptNode.vue";
import axios from 'axios';

const { onConnect, addEdges, setTransform, toObject } = useVueFlow({
  minZoom: 0.2,
  maxZoom: 4,
})


function newTree(){
  // load the default json file into elements
  axios.get(defaultjson)
  .then(function (response) {
    console.log(response);
    elements.value = response.data
  })
}

const elements = ref([])
const defaultjson = './trees/default.json'

function loadDataStart(){
  //call the file open dialog
  const input = document.createElement('input');
  input.type = 'file';
  input.accept = 'application/json';
  input.addEventListener("change", loadData)
  input.dispatchEvent(new MouseEvent("click")); 
}

function loadData(event){
  let fileLocation = URL.createObjectURL(event.target.files[0])
  console.log(fileLocation)
  axios.get(fileLocation)
  .then(function (response) {
    console.log(response);
    elements.value = response.data.map(element => reactive(element));
  })
}

function saveData() {
    const data = JSON.stringify(elements.value)
    const blob = new Blob([data], {type: 'text/plain'})
    const e = document.createEvent('MouseEvents'),
    a = document.createElement('a');
    let nowstring = new Date().toISOString().replace(/:/g, "-") 
    a.download = 'prompttree-'+nowstring+'.json';
    a.href = window.URL.createObjectURL(blob);
    a.dataset.downloadurl = ['text/json', a.download, a.href].join(':');
    e.initEvent('click', true, false, window, 0, 0, 0, 0, 0, false, false, false, false, 0, null);
    a.dispatchEvent(e);
}

function logToObject() {
  return console.log(toObject())
}
function resetTransform() {
  return setTransform({ x: 0, y: 0, zoom: 1 })
}

function onForkPrompt(parent){
  let n_children = elements.value.filter((el) => el.source == parent.id).length
  let new_id = parent.id + '-' + (n_children+1)
  let new_node = {
    id: new_id,
    type: 'prompt',
    data: {
      prompt: parent.data.prompt,
      images: [],
      generating: false,
      width: parent.data.width,
      height: parent.data.height 
    },
    position: { x: parent.position.x + 400, y: parent.position.y + (n_children * 160)},
    targetPosition: Position.Left,
    sourcePosition: Position.Right,
  }
  // make all other nodes unselected
  elements.value.forEach((el) => el.selected = false)
  new_node.selected = true

  // get number of edges in elements
  let e_num =  (elements.value.filter((el) => el.edge).length) + 1
  let new_edge = {
    id: "e"+e_num,
    source: parent.id,
    target: new_id,
    edge: true
  }
  elements.value.push(new_node)
  elements.value.push(new_edge)
  
}


function newNodeUnconnected(){
  // make all other nodes unselected
  elements.value.forEach((el) => el.selected = false)
  // get number of nodes in elements
  let n_num =  (elements.value.filter((el) => el.type == 'prompt').length) + 1
  let new_node = {
    id: "n"+n_num,
    type: 'prompt',
    data: {
      prompt: '',
      images: [],
      generating: false,
      width: 512,
      height: 512
    },
    position: { x: 200, y: 200},
    targetPosition: Position.Left,
    sourcePosition: Position.Right,
  }
  new_node.selected = true
  elements.value.push(new_node)
}

function onUnselect(parent){
  elements.value.find((el) => el.id == parent.id).selected = false
}

function deleteNode(node){
  // check if the node has children connected
  let children = elements.value.filter((el) => el.source == node.id)
  // if it does we wont delete it
  if (children.length > 0){
    console.log('node has children, not deleting')
    onUnselect(node)
    return
  }
  // if it doesnt we will delete it
  console.log('node has no children, deleting')
  //unselect the node
  onUnselect(node)
  // delete the node with id node.id
  elements.value = elements.value.filter((el) => el.id != node.id)
  // detelte the edge with source or target node.id
  elements.value = elements.value.filter((el) => el.source != node.id && el.target != node.id)
  
}

async function genImages(node){
  // console.log('genImages ...', node)
  node.data.generating = true
  // console.log(node)
  // return
  const headers = { 
    'Authorization': 'Bearer '+apiKey.value,
    'Content-Type': 'application/json',
    'Accept': 'application/json',
    'Access-Control-Allow-Origin': '*',
  };
  const body = {
    text_prompts: [
      {
        text: node.data.prompt,
      },
    ],
    cfg_scale: cfg.value,
    clip_guidance_preset: 'FAST_BLUE',
    height: node.data.height,
    width: node.data.width,
    samples: numSamples.value,
    steps: numSteps.value,
  };

  const response = await axios.post(
    apiHost.value+'/v1/generation/'+engineId.value+'/text-to-image', 
    body, 
    {headers:headers}
  ).then(function (response) {
    console.log(response)
    response.data.artifacts.forEach((artifact) => {
      //add the base64 image to the nodes data.images list
      if(artifact.finishReason == 'SUCCESS'){
        // make base64 image into a src url by prepending the data type
        let url = 'data:image/png;base64,'+artifact.base64
        let img = {
          url: url,
          seed: artifact.seed,
        }
        node.data.images.push(img)
      }
    })
  }).catch(function (error) {
    console.log(error);
  }).finally(function(){
    node.data.generating = false
  });
}


function toggleSettings(){
  let settings = document.getElementById('settings')
  if (settings.style.display == 'none'){
    settings.style.display = 'block'
    help.style.display = 'none'
  } else {
    settings.style.display = 'none'
  }
}

function toggleHelp(){
  let help = document.getElementById('help')
  if (help.style.display == 'none'){
    help.style.display = 'block'
    settings.style.display = 'none'
  } else {
    help.style.display = 'none'
  }
}

async function getStabilityEngines(){
  // get the engines from stability.ai and update the engines list
  const headers = { 
    'Authorization': 'Bearer '+apiKey.value,
    'Content-Type': 'application/json',
    'Accept': 'application/json',
    'Access-Control-Allow-Origin': '*',
  };
  const response = await axios.get(
    apiHost.value+"/v1/engines/list", {headers:headers})
    .then(function (response) {
      console.log(response)
      engines.value = response.data
    }).catch(function (error) {
      console.log(error);
    });
}

const apiKey = ref("")
const engineId = ref('stable-diffusion-xl-beta-v2-2-2')
const apiHost = ref('https://api.stability.ai')
const numSamples = ref(4)
const numSteps = ref(30)
const cfg = ref(7)
const width = ref(512)
const height = ref(512)

watch(apiKey, (value) => {localStorage.setItem("apiKey", value);});
watch(engineId, (value) => {localStorage.setItem("engineId", value);});
watch(apiHost, (value) => {localStorage.setItem("apiHost", value);});
watch(numSamples, (value) => {localStorage.setItem("numSamples", value);});
watch(numSteps, (value) => {localStorage.setItem("numSteps", value);});
watch(cfg, (value) => {localStorage.setItem("cfg", value);});
watch(width, (value) => {localStorage.setItem("width", value);});
watch(height, (value) => {localStorage.setItem("height", value);});

onConnect(addEdges)

onMounted(() => {
  // load the default json file into elements
  axios.get(defaultjson)
  .then(function (response) {
    console.log(response);
    elements.value = response.data
  })
  //get the settings from localstorage
  apiKey.value = localStorage.getItem("apiKey") || "";
  engineId.value = localStorage.getItem("engineId") || "stable-diffusion-xl-beta-v2-2-2";
  apiHost.value = localStorage.getItem("apiHost") || "https://api.stability.ai";
  numSamples.value = parseInt(localStorage.getItem("numSamples")) || 4;
  numSteps.value = parseInt(localStorage.getItem("numSteps")) || 30;
  cfg.value = parseInt(localStorage.getItem("cfg")) || 7;
  width.value = parseInt(localStorage.getItem("width")) || 512;
  height.value = parseInt(localStorage.getItem("height")) || 512;
})





</script>

<template>
    <!-- <div id="nav">
      prompt tree
    </div> -->
    <VueFlow v-model="elements" 
      :default-edge-options="{ type: 'smoothstep' }"
      :default-viewport="{ zoom: 0.75 }"
      :fit-view-on-init="false"
      :snap-to-grid="true"
      class="vue-flow-outer"
      >
      <template #node-prompt="props">
        <PromptNode 
          v-bind="props"
          @forkPrompt="onForkPrompt(props)"
          @unselect="onUnselect(props)"
          @generate="genImages(props)"
          @deleteNode="deleteNode(props)"
        />
      </template>

      <Background />
      <Panel :position="PanelPosition.TopLeft" style="display: flex; gap: 5px">
        <button @click="saveData">save</button>
        <button @click="loadDataStart">load</button>
        <button @click="newTree">new</button>
        <!-- <button @click="newNodeUnconnected">add node</button> -->
      </Panel>
      <Panel :position="PanelPosition.TopRight" style="display: flex; gap: 5px">
        <button @click="toggleHelp">?</button>
        <button @click="toggleSettings">settings</button>
        <a href="https://github.com/isaac-art/prompttree" target="_blank">
          <button><i class=" white fa fa-github" aria-hidden="true"></i></button>
          </a>
      </Panel>

      <Controls />
    </VueFlow>

    

    <div id="help" class="modal">
      <div class="settings_row">
        Type a prompt in a Node, on 'Enter' key or 'gen' button the images will be generated and the prompt locked.
        <br><br>        
        Once the images are ready the Node will be updated and then has two options 'regen' and 'fork'.
        <br><br>
        'regen' will make another batch of images for this Node.
        <br><br>
        'fork' will make a new Node with the same prompt as this Node, ready to be edited.
        <br><br>
        'save' will export a .json file with the prompts and images stored in the tree markup.
        <br><br>
        'load' will import a .json file with the prompts and images stored in the tree markup.
        <br><br>
        'new' will clear the tree and start a new one.
        <br><br>
        'settings' will open the settings modal to change api key and other stable diffusion settings.
        <br><br>
        '?' will toggle this help modal.
      </div>
      <div class="settings_row">
        <button class="settings_btn_close" @click="toggleHelp">close</button>
      </div>
    </div>

    <div id="settings" class="modal">

      <div class="settings_row">
        <label for="api_key_input">stability api key 
          <small><a href="https://platform.stability.ai/docs/getting-started/authentication" target="_blank">see docs</a></small>
        </label>
        <input id="api_key_input" name="api_key_input" 
            type="text" v-model="apiKey" class="api_key_input" />
      </div>
      <div class="settings_row">
        your api key is only stored in browser localstorage and is only used to make frontend requests to the stability.ai api
      </div>
      <div class="settings_row">
        <label for="engine_id_input">engine id</label>
        <input id="engine_id_input" name="engine_id_input" 
            type="text" v-model="engineId" class="api_key_input" />
      </div>

      <div class="settings_row">
        <label for="num_samples_input">samples</label>
        <input type="number" min="1" max="8"
          v-model="numSamples" name="num_samples_input" />
      </div>

      <div class="settings_row">
        <button class="settings_btn_close" @click="toggleSettings">close</button>
      </div>
    </div>
</template>

<style >
*{
  font-family: 'IBM Plex Mono', monospace;
}
.white{
  color: white;
}
#credits{
  position: fixed;
  bottom:0px;
  right: 10px;
  font-size: 0.75rem;
  text-align: right;
  color: #333
}
#credits a{
  color: #333;
}
.modal{
  /* div centered in screen, fixed */
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  /* styling */
  width: 50vw;
  min-width: 420px;;
  height: 25vh;
  min-height: 320px;
  /* display: none; */ /* Show by default */ 
  background: #333;
  color: white;
  padding: 20px;
  border-radius: 3px;
}
#help{
  display: none;
  min-height: 600px;
  min-width: 500px;
}
#settings{
  display: block;
}
#settings small{
  float: right;
}
#settings a{
  color: azure;
}
.settings_btn_close{
  float: right;
}
.settings_row{
  margin: 10px;
  margin-bottom:20px;
}
.settings_row label{
  display: inline-block;
  width: 100%;
}
.settings_row input{
  width: 100%;
  padding: 5px;
  border-radius: 3px;
  border: 1px solid white;
  background: #333;
  color: white;
}

.api_key_input{
  width: 420px;
}
#nav{
  height: 3vh;
  margin: 0 auto;
  padding:10px; 
  background-color: #333;
  color: whitesmoke;
}
.vue-flow-outer{
  height: 100vh;
}
button {
  text-transform: lowercase;
  background: #333;
  color: white;
  border: none;
  padding: 5px 10px;
  border-radius: 3px;
  cursor: pointer;
  font-size: 14px;
  font-family: 'IBM Plex Mono', monospace;
  outline: none;
  transition: background 0.3s ease;
}
button:hover {
  background: #555;
}
button:active {
  background: #777;
} 
</style>
