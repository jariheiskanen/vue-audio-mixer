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
</script>

<template>
  <div class="layout-wrapper">
    <div class="main-section">
      <!--main section here-->
    </div>
    <div class="channel-wrapper">
      <AudioChannel v-for="i in channelCount" :key="i" :file="audioFiles[i-1] || null" @file-added="handleFileAdded"/>
    </div>
  </div>
</template>

<style scoped>

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
