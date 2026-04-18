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
  {id: 1, duration: 0, start: 0, volume: 1, trimStart: 0, trimEnd: 0 },
  {id: 2, duration: 0, start: 0, volume: 1, trimStart: 0, trimEnd: 0 },
  {id: 3, duration: 0, start: 0, volume: 1, trimStart: 0, trimEnd: 0 },
  {id: 4, duration: 0, start: 0, volume: 1, trimStart: 0, trimEnd: 0 }
]); //length of each audio file and starting point in s
const ZOOM_LEVELS = [0.1, 0.25, 0.5, 0.75, 1, 1.25, 1.5, 2, 3];
const zoomIndex = ref(ZOOM_LEVELS.indexOf(1)); // start at 100%
const scrollLeft = ref(0);

// longest visible timeline
const MINIMUM_TIMELINE = 10;
const timelineDuration = computed(() =>
  Math.max(...channels.value.map(c => c.start + c.duration), MINIMUM_TIMELINE)
);

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

//update duration
function updateDuration(index, duration) 
{
  channels.value[index] = {...channels.value[index],duration};
  //init to default values
  channels.value[index].trimStart = 0;
  channels.value[index].trimEnd = duration;
}

//update starting point (from dragging)
function updateStart(index, start)
{
  channels.value[index] = {...channels.value[index],start};
}

function updateVolume(index, volume)
{
  channels.value[index] = {...channels.value[index],volume};
}

function handleWheel(e)
{
  if (!e.target.closest('.audio-visual-wrap')) return;
  e.preventDefault();

  if (e.deltaY < 0)
  {
    // zoom in
    zoomIndex.value = Math.min(ZOOM_LEVELS.length - 1, zoomIndex.value + 1);
  }
  else
  {
    // zoom out
    zoomIndex.value = Math.max(0, zoomIndex.value - 1);
  }
}

function onGraphScroll(scroll)
{
  scrollLeft.value = scroll;
}

//combine audio channels
async function buildMix()
{
  //init
  const sampleRate = 44100;
  const duration = timelineDuration.value;

  const offlineCtx = new OfflineAudioContext(
    2,
    duration * sampleRate,
    sampleRate
  );

  //process each channel
  for (let i = 0; i < channels.value.length; i++)
  {
    const file = audioFiles.value[i];
    if (!file) continue;

    const arrayBuffer = await file.arrayBuffer();
    const buffer = await offlineCtx.decodeAudioData(arrayBuffer);

    const source = offlineCtx.createBufferSource();
    source.buffer = buffer;

    // create gain node for volume
    const gainNode = offlineCtx.createGain();
    gainNode.gain.value = channels.value[i].volume;

    source.connect(gainNode);
    gainNode.connect(offlineCtx.destination);

    //logic for cut files
    const ch = channels.value[i];
    const startTime = ch.start;
    const offset = Math.max(0, ch.trimStart || 0);

    const playDuration = Math.max(
      0,
      Math.min(
        (ch.trimEnd ?? buffer.duration) - offset,
        buffer.duration - offset
      )
    );

    source.start(startTime, offset, playDuration);
  }

  const renderedBuffer = await offlineCtx.startRendering();
  return renderedBuffer;
}

//wav file format
function bufferToWav(buffer)
{
  const numOfChan = buffer.numberOfChannels;
  const length = buffer.length * numOfChan * 2;
  const bufferOut = new ArrayBuffer(44 + length);
  const view = new DataView(bufferOut);

  let offset = 0;

  function writeString(str) {
    for (let i = 0; i < str.length; i++)
      view.setUint8(offset++, str.charCodeAt(i));
  }

  // WAV header
  writeString('RIFF');
  view.setUint32(offset, 36 + length, true); offset += 4;
  writeString('WAVE');
  writeString('fmt ');
  view.setUint32(offset, 16, true); offset += 4;
  view.setUint16(offset, 1, true); offset += 2;
  view.setUint16(offset, numOfChan, true); offset += 2;
  view.setUint32(offset, buffer.sampleRate, true); offset += 4;
  view.setUint32(offset, buffer.sampleRate * 2 * numOfChan, true); offset += 4;
  view.setUint16(offset, numOfChan * 2, true); offset += 2;
  view.setUint16(offset, 16, true); offset += 2;
  writeString('data');
  view.setUint32(offset, length, true); offset += 4;

  // interleave
  const channels = [];
  for (let i = 0; i < numOfChan; i++)
    channels.push(buffer.getChannelData(i));

  let sample = 0;
  while (sample < buffer.length)
  {
    for (let i = 0; i < numOfChan; i++)
    {
      let s = Math.max(-1, Math.min(1, channels[i][sample]));
      view.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7FFF, true);
      offset += 2;
    }
    sample++;
  }

  return new Blob([view], { type: 'audio/wav' });
}

//download file
function downloadBlob(blob)
{
  const url = URL.createObjectURL(blob);

  const a = document.createElement('a');
  a.href = url;
  a.download = 'SimpleAudioMixer.wav';
  a.click();

  URL.revokeObjectURL(url);
}

//process and download
async function exportAudio()
{
  const renderedBuffer = await buildMix(); // steps above
  const wav = bufferToWav(renderedBuffer);
  downloadBlob(wav);
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
      <button @click="exportAudio">Download</button>
    </div>
    <div class="channel-wrapper">
      <AudioChannel v-for="i in channelCount" :key="i" :file="audioFiles[i-1] || null" :timeline-duration="timelineDuration" :start="channels[i-1].start" :duration="channels[i-1].duration" @duration="updateDuration(i-1, $event)" :volume="channels[i-1].volume" @update:volume="updateVolume(i-1, $event)" @file-added="handleFileAdded" @update:start="updateStart(i-1,$event)" :zoom-level="zoom_level" :scroll-left="scrollLeft" @scroll:graph="onGraphScroll" @update:trimStart="channels[i-1].trimStart = $event" @update:trimEnd="channels[i-1].trimEnd = $event"/>
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
