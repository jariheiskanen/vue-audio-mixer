<!--
npm run dev
npm run build
npm run deploy
-->

<script setup>
import {ref, computed, onMounted, onUnmounted} from 'vue';
import AudioChannel from './components/AudioChannel.vue'

const audioFiles = ref([]);
const MAX_FILES = 4;
const channels = ref([
  {id: 1, duration: 0, start: 0 },
  {id: 2, duration: 0, start: 0 },
  {id: 3, duration: 0, start: 0 },
  {id: 4, duration: 0, start: 0 }
]); //length of each audio file and starting point in s
const ZOOM_LEVELS = [0.1, 0.25, 0.5, 0.75, 1, 1.25, 1.5, 2, 3];
const zoomIndex = ref(ZOOM_LEVELS.indexOf(1)); // start at 100%
const scrollLeft = ref(0);

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
const MINIMUM_TIMELINE = 10;
const timelineDuration = computed(() =>
  Math.max(...channels.value.map(c => c.start + c.duration), MINIMUM_TIMELINE)
);

//update duration
function updateDuration(index, duration) 
{
  channels.value[index] = {...channels.value[index],duration};
}

//update starting point (from dragging)
function updateStart(index, start)
{
  channels.value[index] = {...channels.value[index],start};
}

function handleWheel(e)
{
  if (!e.target.closest('.audio-visual-wrap')) return;
  e.preventDefault();

  

  const zoomFactor = 1.1;
  if (e.deltaY < 0)
  {
    // zoom in
    //zoom_level.value *= zoomFactor;
    zoomIndex.value = Math.min(ZOOM_LEVELS.length - 1, zoomIndex.value + 1);
  }
  else
  {
    // zoom out
    //zoom_level.value /= zoomFactor;
    zoomIndex.value = Math.max(0, zoomIndex.value - 1);
  }

  // clamp zoom
  //zoom_level.value = Math.min(5, Math.max(0.2, zoom_level.value));
}

function onGraphScroll(scroll)
{
  scrollLeft.value = scroll;
}

const zoom_level = computed(() => ZOOM_LEVELS[zoomIndex.value]);

onMounted(() => {
  window.addEventListener('wheel', handleWheel, { passive: false });
})

onUnmounted(() => {
  window.removeEventListener('wheel', handleWheel);
})

</script>

<template>
  <div class="layout-wrapper">
    <div class="main-section">
      <!--main section here-->
      ZOOM: {{ Math.round(zoom_level*100)+'%' }}
    </div>
    <div class="channel-wrapper">
      <AudioChannel v-for="i in channelCount" :key="i" :file="audioFiles[i-1] || null" :timeline-duration="timelineDuration" :start="channels[i-1].start" :duration="channels[i-1].duration" @duration="updateDuration(i-1, $event)" @file-added="handleFileAdded" @update:start="updateStart(i-1,$event)" :zoom-level="zoom_level" :scroll-left="scrollLeft" @scroll:graph="onGraphScroll"/>
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
