<!--
npm run dev
npm run build
npm run deploy
-->

<script setup>
import {ref, computed} from 'vue';
import AudioChannel from './components/AudioChannel.vue'

const audioFiles = ref([]);
const MAX_FILES = 4;
const channels = ref([
  {id: 1, duration: 0, start: 0 },
  {id: 2, duration: 0, start: 0 },
  {id: 3, duration: 0, start: 0 },
  {id: 4, duration: 0, start: 0 }
]); //length of each audio file and starting point in s

//adds files to an array
function handleFileAdded(file) 
{
  if(audioFiles.value.length >= MAX_FILES) return;
  audioFiles.value.push(file);
}

//limits channels to MAX_FILES
const channelCount = computed(() => {
  return Math.min(audioFiles.value.length + 1, MAX_FILES);
});

// longest visible timeline
const timelineDuration = computed(() =>
  Math.max(...channels.value.map(c => c.start + c.duration), 0)
);

function updateDuration(index, duration) 
{
  channels.value[index].duration = duration;
}
</script>

<template>
  <div class="layout-wrapper">
    <div class="main-section">
      <!--main section here-->
    </div>
    <div class="channel-wrapper">
      <AudioChannel v-for="i in channelCount" :key="i" :file="audioFiles[i-1] || null" :timeline-duration="timelineDuration" :start="channels[i-1].start" :duration="channels[i-1].duration" @duration="updateDuration(i-1, $event)" @file-added="handleFileAdded"/>
    </div>
  </div>
</template>

<style scoped>

.layout-wrapper
{
  font-family: system-ui, sans-serif;
}

.main-section
{
  min-height: 545px;
}

.channel-wrapper
{
  margin-top: auto;
  display: flex;
  flex-direction: column;
  position: absolute;
  bottom: 0;
  width: 100%;
}
</style>
