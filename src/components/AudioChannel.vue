<!--
TODO:
- graph timer marker scaling dynamically
- precision setting eg. (1.52s)
- set timers by typing
- fade audio in/out
- better play button
- change speed
- save in different formats, wav done
- move from lamejs to ffmpeg.wasm?

-->

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue';
import lamejs from 'lamejs';

/*globals need to be defined manually because its an old library*/
import MPEGMode from 'lamejs/src/js/MPEGMode';
import Lame from 'lamejs/src/js/Lame';
import BitStream from 'lamejs/src/js/BitStream';
window.MPEGMode = MPEGMode;
window.Lame = Lame;
window.BitStream = BitStream;



const emit = defineEmits(['file-added']);

const fileRef = ref(null);
const AudioRef = ref(null);
const CanvasRef = ref(null);
const ContainerRef = ref(null);
const audioCtx = new AudioContext();
const gainNode = ref(null);
const AudioBuffer = ref(null);

const showUpload = ref(true); //shows upload button when audio file hasn't been added
const audioPlaying = ref(false); //true when audio file is being played
const fileAdded = ref(false); //file added
const fileName = ref(null); //name of the selected file

const playProgress = ref(0); //progress of the audio file marker in %
const audioDuration = ref(0); //duration of audio file in s
const audioCurrent = ref(0); //current timer of audio in s
const audioVolume = ref(1); //volume of audio
const loopEnabled = ref(false); //enabled looping setting

//trim
const trimStart = ref(0); //starting point in s
const trimEnd = ref(0); //ending point in s
const dragHandle = ref(null); //which handle is being dragged
const disabledClick = ref(false); //click disabled for seeking during drag

//mouse hover
const hoverSeekX = ref(0);
const hoverVisible = ref(false);
const hoverSeekTime = ref(0);
const hoveringHandle = ref(false); //hovering drag handle

//cut timers
const cutStart = ref(0); //start of cut file in s
const cutEnd = ref(0); //end of cut file in s

let animationId = null; //used for audio progress animation
let animationId2 = null; //used for timer progress
let animationId3 = null; //used for audio play ending for trim end

//process uploaded file
async function decodeFile(file) 
{
  const arrayBuffer = await file.arrayBuffer();
  return await audioCtx.decodeAudioData(arrayBuffer);
}

//get volume data for file at set sample rate
function getVolumeData(audioBuffer, samples = 500, startSample, endSample) 
{
    const channelData = audioBuffer.getChannelData(0); // mono
    const visibleData = channelData.slice(startSample, endSample);
    const blockSize = Math.floor(visibleData.length / samples);
    const volumes = [];

    for (let i=0; i<samples; i++) 
    {
        let sum = 0;
        const start = i*blockSize;

        for (let j=0; j<blockSize; j++) 
        {
            const sample = visibleData[start + j];
            sum += sample*sample;
        }

        const rms = Math.sqrt(sum/blockSize);
        volumes.push(rms);
    }

    return volumes;
}

//smooths graph to have less spikes, higher radius = smoother
function smooth(data, radius = 1) {
  return data.map((_, i) => {
    let sum = 0;
    let count = 0;

    for (let j = -radius; j <= radius; j++) {
      const idx = i + j;
      if (idx >= 0 && idx < data.length) {
        sum += data[idx];
        count++;
      }
    }
    return sum / count;
  });
}

//normalizes graph to fit the canvas
function normalize(data) 
{
    const max = Math.max(...data);
    return data.map(v => v / max);
}

//sharpens the graph
function setupCanvas(canvas) 
{
    const dpr = window.devicePixelRatio || 1;
    const rect = canvas.getBoundingClientRect();

    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;

    const ctx = canvas.getContext('2d');
    ctx.scale(dpr, dpr);

    return ctx;
}

//draws audio graph for uploaded file
function drawVolumeGraph(canvas, data) 
{
    //init canvas
    const ctx = setupCanvas(canvas);
    const width = canvas.width;
    const height = canvas.height;

    //clear canvas
    ctx.clearRect(0, 0, width, height);
    
    //draw a line for each sample, close and fill the graph
    ctx.beginPath();
    ctx.moveTo(0, height);

    const barWidth = width / data.length;
    data.forEach((value, i) => {
        const barHeight = value * height;
        const x = i * barWidth;
        const y = height - barHeight;

        ctx.lineTo(x, y);
    });

    ctx.lineTo(width, height);
    ctx.closePath();
    ctx.fillStyle = "#0da90d";
    ctx.fill();

    drawMarkers(ctx, width, height);
}

//draw timer markers
function drawMarkers(ctx, width, height) 
{
    const interval = 1; //interval of markers in seconds
    const min_spacing = 50; //minimum allowed distance between markers

    //line styling
    ctx.font = '10px Arial';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'top';
    ctx.fillStyle = 'white';
    ctx.lineStyle = 'white';

    let last_drawn_x = 0 - min_spacing;
    for(let time=0; time<=audioDuration.value; time+=interval) 
    {
        const x = (time/audioDuration.value)*width;
        if(last_drawn_x + min_spacing <= x)
        {
            ctx.strokeStyle = 'rgba(255,255,255,0.8)';
            ctx.lineWidth = 1;
            // Draw tick line
            ctx.beginPath();
            ctx.moveTo(x, height - 7);
            ctx.lineTo(x, height);
            ctx.stroke();

            ctx.strokeStyle = 'black';
            ctx.lineWidth = 2;
            // Draw label
            ctx.strokeText(time, x + 2, height - 18);
            ctx.fillText(time, x + 2, height - 18);
            last_drawn_x = x;
        }
    }
}

//file upload
async function handleFile(e) 
{
    const file = e.target.files[0];
    fileRef.value = file;
    if (!file) return;

    showUpload.value = false; //hide upload icon
    fileAdded.value = true;
    fileName.value = file.name;

    const fileURL = URL.createObjectURL(file);
    AudioRef.value.src = fileURL;
    AudioRef.value.load();

    //process, normalize and smooth value before drawing it to canvas
    const audioBuffer = await decodeFile(file);
    const volumeData = getVolumeData(audioBuffer, 600, 0, audioBuffer.length);
    const normalized = normalize(volumeData);
    const smoothed = smooth(normalized);
    AudioBuffer.value = audioBuffer;

    drawVolumeGraph(CanvasRef.value, smoothed);
    initAudioGraph();

    //emit file to parent
    emit('file-added', file);
}

//change volume
function changeVolume(e)
{
    gainNode.value.gain.value = Number(e.target.value);
}

//plays/pauses audio
function togglePlay()
{
    if(!audioPlaying.value)
    {
        AudioRef.value.play();
        monitorPlayback();
        audioPlaying.value = true;
    }
    else
    {
        AudioRef.value.pause();
        audioPlaying.value = false;
    }
}

//animate audio play bar
function animate() 
{
  const audio = AudioRef.value;
  const container = ContainerRef.value;
  if (!audio || !container || audio.paused) return;

  playProgress.value = ((audio.currentTime - cutStart.value) / audioDuration.value) * container.clientWidth;

  animationId = requestAnimationFrame(animate);
}

//updates current time of audio file
function updateTime() 
{
  if (!AudioRef.value) return;
  audioCurrent.value = AudioRef.value.currentTime - cutStart.value;
  animationId = requestAnimationFrame(updateTime);
}

//start animation
function startPlay()
{
    //progress bar
    cancelAnimationFrame(animationId);
    animationId = requestAnimationFrame(animate);

    //timer
    cancelAnimationFrame(animationId2);
    animationId2 = requestAnimationFrame(updateTime);
}

//on pause
function pausePlay()
{
    cancelAnimationFrame(animationId);
    cancelAnimationFrame(animationId2);
}

//skip through audio by clicking
function seek(e) {
    if(!fileAdded.value) return;
    if(disabledClick.value) return;
    const audio = AudioRef.value;
    const container = ContainerRef.value;
    const rect = container.getBoundingClientRect();

    const clickX = e.clientX - rect.left;
    let percent = clickX / rect.width;

    const min = startPercent.value / 100;
    const max = endPercent.value / 100;

    //value is set between min and max based on trim sliders
    percent = Math.min(Math.max(percent, min), max);

    audio.currentTime = cutStart.value + percent * audioDuration.value;
    playProgress.value = ((audio.currentTime - cutStart.value) / audioDuration.value) * container.clientWidth;
    audioCurrent.value = audio.currentTime - cutStart.value;
}

//seek hover effect
function hoverSeek(e)
{
    const rect = CanvasRef.value.getBoundingClientRect();
    const x = e.clientX - rect.left;

    hoverSeekX.value = x;
    hoverVisible.value = true;

    const percent = x / rect.width;
    hoverSeekTime.value = percent * audioDuration.value;
}

//mouse exit hover zone
function hoverLeave()
{
  hoverVisible.value = false;
}

//sets total length of audio
function setDuration(e)
{
    audioDuration.value = e.target.duration;
    trimEnd.value = audioDuration.value;
    cutEnd.value = audioDuration.value;
}

//Format seconds to mm:ss.M
function formatTime(sec) {
    sec = Number(sec);
    if(sec >= 0)
    {
        const minutes = Math.floor(sec / 60);
        const seconds = Math.floor(sec % 60);
        const tenths = Math.floor((sec % 1) * 10);
        return `${minutes}:${seconds.toString().padStart(2,'0')}.${tenths}`;
    }
    else
    {
        return "0:00.0";
    }
    

}

//start drag on trim handle
function startDrag(type) 
{
  dragHandle.value = type;
  disabledClick.value = true;
  window.addEventListener('mousemove', onDrag);
  window.addEventListener('mouseup', stopDrag);
}

//drag logic
function onDrag(e) 
{
  if (!dragHandle.value) return;

  const container = ContainerRef.value;
  const rect = container.getBoundingClientRect();

  let percent = (e.clientX - rect.left) / rect.width;
  percent = Math.min(Math.max(0, percent), 1);

  const newTime = percent * audioDuration.value;

  if (dragHandle.value === 'start') 
  {
    trimStart.value = Math.min(newTime, trimEnd.value - 0.1);

    //keep current audio marker inside trim limits
    if(trimStart.value > audioCurrent.value)
    {
        AudioRef.value.currentTime = trimStart.value;
        audioCurrent.value = trimStart.value;
        playProgress.value = (audioCurrent.value / audioDuration.value) * container.clientWidth;
    }
  }

  if (dragHandle.value === 'end') 
  {
    trimEnd.value = Math.max(newTime, trimStart.value + 0.1);

    if(trimEnd.value < audioCurrent.value)
    {
        audioCurrent.value = trimEnd.value;
        playProgress.value = playProgress.value = (audioCurrent.value / audioDuration.value) * container.clientWidth;
    }
  }
}

//stop drag event
function stopDrag() 
{
    dragHandle.value = null;
    window.removeEventListener('mousemove', onDrag);
    window.removeEventListener('mouseup', stopDrag);
    
    //enable click for seeking after 5ms
    setTimeout(() => {
        disabledClick.value = false;
    }, 5)
}

//stops audio playing at ending trim marker
function monitorPlayback() {
  const audio = AudioRef.value;
  if(!audio) return;

  if(audio.currentTime >= trimEnd.value+cutStart.value) 
  {
    //if looping is enabled, don't pause at the end
    if(loopEnabled.value)
    {
        audio.currentTime = trimStart.value+cutStart.value;
    }
    else
    {
        audio.pause();
        audio.currentTime = trimStart.value+cutStart.value;
        playProgress.value = ((audio.currentTime - cutStart.value) / audioDuration.value) * ContainerRef.value.clientWidth;
        audioCurrent.value = 0;
        audioPlaying.value = false;
        cancelAnimationFrame(animationId3);
        return;
    }    
  }

  animationId3 = requestAnimationFrame(monitorPlayback);
}

//cut audio file for editing
//doesn't actually cut the file, just sets cutStart and cutEnd that are treated visually as cut file
async function cutFile()
{
    
    //set values
    cutStart.value = trimStart.value;
    cutEnd.value = trimEnd.value;
    audioDuration.value = cutEnd.value - cutStart.value;

    redrawCanvas(trimStart.value, trimEnd.value);

    AudioRef.value.currentTime = cutStart.value;
    playProgress.value = 0;
    audioCurrent.value = 0;
    trimStart.value = 0;
    trimEnd.value = audioDuration.value;    
}

//initialize audio context for volume editing
function initAudioGraph() {
  const ctx = new AudioContext();
  const source = ctx.createMediaElementSource(AudioRef.value);
  gainNode.value = ctx.createGain();

  source.connect(gainNode.value);
  gainNode.value.connect(ctx.destination);
}

//redraws canvas
function redrawCanvas(start, end)
{
    if (!AudioBuffer.value) return;
    const sampleRate = AudioBuffer.value.sampleRate;
    const startSample = Math.floor(start * sampleRate);
    const endSample = Math.floor(end * sampleRate);

    const volumeData = getVolumeData(AudioBuffer.value, 600, startSample, endSample);
    const normalized = normalize(volumeData);
    const smoothed = smooth(normalized);
    drawVolumeGraph(CanvasRef.value, smoothed); 
}

//downloads file based on current cut limits
function downloadFile(type)
{
    const trimmed = trimAudioBuffer();
    let blob = null;
    let download_name = null;
    switch(type)
    {
        case "wav":
            blob = convertWav(trimmed);
            download_name = fileName.value.replace(/\.[^/.]+$/, '') + '-edit.wav';
            break;
        case "mp3":
            blob = convertMp3(trimmed);
            download_name = fileName.value.replace(/\.[^/.]+$/, '') + '.mp3';
            break;
        default:
    }

    //download file
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = download_name;
    a.click();
    URL.revokeObjectURL(url);
}

//creates audio buffer based on cut limits
function trimAudioBuffer() 
{
  const sampleRate = AudioBuffer.value.sampleRate;
  const startSample = Math.floor(cutStart.value * sampleRate);
  const endSample = Math.floor(cutEnd.value * sampleRate);
  const frameCount = endSample - startSample;

  const trimmedBuffer = audioCtx.createBuffer(AudioBuffer.value.numberOfChannels,frameCount,sampleRate);

  for(let ch=0; ch<AudioBuffer.value.numberOfChannels; ch++) 
  {
    const oldData = AudioBuffer.value.getChannelData(ch);
    const newData = trimmedBuffer.getChannelData(ch);

    for (let i = 0; i < frameCount; i++) 
    {
      //apply volume boost and clamp to avoid distortion overflow
      const boosted = oldData[startSample+i]*audioVolume.value;
      newData[i] = Math.max(-1, Math.min(1, boosted));
    }
  }

  return trimmedBuffer;
}

//converts audio buffer into wav blob
function convertWav(buffer)
{
  const numChannels = buffer.numberOfChannels;
  const sampleRate = buffer.sampleRate;
  const length = buffer.length*numChannels*2 + 44;
  const view = new DataView(new ArrayBuffer(length));

  let offset = 0;

  function writeString(str) 
  {
    for (let i = 0; i < str.length; i++) 
    {
      view.setUint8(offset++, str.charCodeAt(i));
    }
  }

  // WAV header
  writeString('RIFF');
  view.setUint32(offset, length-8, true); offset += 4;
  writeString('WAVE');
  writeString('fmt ');
  view.setUint32(offset, 16, true); offset += 4;
  view.setUint16(offset, 1, true); offset += 2;
  view.setUint16(offset, numChannels, true); offset += 2;
  view.setUint32(offset, sampleRate, true); offset += 4;
  view.setUint32(offset, sampleRate * numChannels * 2, true); offset += 4;
  view.setUint16(offset, numChannels * 2, true); offset += 2;
  view.setUint16(offset, 16, true); offset += 2;
  writeString('data');
  view.setUint32(offset, buffer.length * numChannels * 2, true); offset += 4;

  // PCM data
  for (let i = 0; i < buffer.length; i++) 
  {
    for (let ch = 0; ch < numChannels; ch++) 
    {
      let sample = buffer.getChannelData(ch)[i];
      sample = Math.max(-1, Math.min(1, sample));
      view.setInt16(offset, sample * 0x7fff, true);
      offset += 2;
    }
  }

  return new Blob([view], { type: 'audio/wav' });
}

//converts audio buffer into mp3 blob
function convertMp3(buffer, bitrate = 128) 
{
  const numChannels = buffer.numberOfChannels;
  const sampleRate = buffer.sampleRate;
  const samples = buffer.getChannelData(0); // mono (simplest)

  // convert Float32 → Int16
  const pcm = new Int16Array(samples.length);
  for (let i = 0; i < samples.length; i++) 
  {
    const s = Math.max(-1, Math.min(1, samples[i]));
    pcm[i] = s * 0x7fff;
  }

  const mp3encoder = new lamejs.Mp3Encoder(1, sampleRate, bitrate);
  const blockSize = 1152;
  const mp3Data = [];

  for (let i = 0; i < pcm.length; i += blockSize) 
  {
    const chunk = pcm.subarray(i, i + blockSize);
    const mp3buf = mp3encoder.encodeBuffer(chunk);
    if (mp3buf.length) mp3Data.push(mp3buf);
  }

  const end = mp3encoder.flush();
  if (end.length) mp3Data.push(end);

  return new Blob(mp3Data, { type: 'audio/mp3' });
}

//trim handle positions
const startPercent = computed(() =>
  (trimStart.value / audioDuration.value) * 100
);

const endPercent = computed(() =>
  (trimEnd.value / audioDuration.value) * 100
);

//visual volume number
const volumeDisplay = computed(() => {
  return Math.round(audioVolume.value * 100) + "%";
});

const showHoverBar = computed(() => {
  return fileAdded.value && hoverVisible.value &&!dragHandle.value && !hoveringHandle.value && hoverSeekTime.value > trimStart.value && hoverSeekTime.value < trimEnd.value;
})

//window resize
let resizeObserver;
onMounted(() => {
  resizeObserver = new ResizeObserver(() => {
    redrawCanvas(cutStart.value, cutEnd.value);
  });

  resizeObserver.observe(CanvasRef.value);
});

onBeforeUnmount(() => {
  resizeObserver.disconnect();
});

</script>

<template>
    <div class="channel">
        <div class="control-bar" v-if="fileAdded">
            <div class="left-wrapper">
                <div class="audio-info">
                    <div>{{fileName}}</div>
                    <div>{{formatTime(audioDuration)}}</div>
                </div>
            </div>

            <div class="center-wrapper">
                <span>{{formatTime(audioCurrent)}}</span>
                <div class="play-audio-wrapper" @click="togglePlay">
                    <svg class="play" v-if="!audioPlaying" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 50 50">
                        <path d="M5.716 49.715a2.272 2.272 0 0 0 2.306 -0.061l36.364 -22.727a2.273 2.273 0 0 0 0 -3.855L8.023 0.345A2.273 2.273 0 0 0 4.545 2.273v45.455a2.273 2.273 0 0 0 1.171 1.988"/>
                    </svg>
                    <svg v-else viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg">
                        <path d="M4.694 1.758A0.787 0.787 0 0 0 3.906 2.545v14.909A0.787 0.787 0 0 0 4.694 18.242h2.566A0.787 0.787 0 0 0 8.047 17.455V2.545A0.787 0.787 0 0 0 7.259 1.758zm8.047 0A0.787 0.787 0 0 0 11.953 2.545v14.909A0.787 0.787 0 0 0 12.741 18.242h2.566A0.787 0.787 0 0 0 16.094 17.455V2.545A0.787 0.787 0 0 0 15.306 1.758z"/>
                    </svg>
                </div>                
            </div>

            <div class="right-wrapper">
                <div class="button-wrap">
                    <div class="square-button" @click="downloadFile('mp3')" title="Download (.mp3)">
                        <!-- License: PD. Made by icons8: https://github.com/icons8/windows-10-icons -->
                        <svg version="1.1" id="Layer_1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" viewBox="0 0 32 32" enable-background="new 0 0 32 32" xml:space="preserve">
                            <line fill="none" stroke="#000000" stroke-width="3" stroke-miterlimit="10" x1="25" y1="28" x2="7" y2="28"/>
                            <line fill="none" stroke="#000000" stroke-width="2" stroke-miterlimit="10" x1="16" y1="23" x2="16" y2="4"/>
                            <polyline fill="none" stroke="#000000" stroke-width="2" stroke-miterlimit="10" points="9,16 16,23 23,16 "/>
                        </svg>
                    </div>
                    <div class="square-button" @click="downloadFile('wav')" title="Download (.wav)">
                        <!-- License: PD. Made by icons8: https://github.com/icons8/windows-10-icons -->
                        <svg version="1.1" id="Layer_1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" viewBox="0 0 32 32" enable-background="new 0 0 32 32" xml:space="preserve">
                            <line fill="none" stroke="#000000" stroke-width="3" stroke-miterlimit="10" x1="25" y1="28" x2="7" y2="28"/>
                            <line fill="none" stroke="#000000" stroke-width="2" stroke-miterlimit="10" x1="16" y1="23" x2="16" y2="4"/>
                            <polyline fill="none" stroke="#000000" stroke-width="2" stroke-miterlimit="10" points="9,16 16,23 23,16 "/>
                        </svg>
                    </div>
                    <div class="square-button" @click="cutFile" title="Cut">
                        <svg xmlns="http://www.w3.org/2000/svg" width="20px" height="20px" viewBox="0 0 512 372">
                            <path d="M54.5 1.6C33.9 6.2 18.1 17.5 11.6 32.3c-5.8 13.1-5.6 25.3.9 38.2 4.4 9 10 15.4 18.6 21.5 7.7 5.4 13.4 7.6 53 20.4 20.8 6.8 30 10.4 41.5 16.1 19.7 9.9 29.9 17.4 47.7 35.3l14.8 14.8-12.5 12.2c-19.4 18.8-34.3 29.1-55.9 38.2-6.2 2.6-25.9 9.5-43.8 15.4-33.6 11-37.6 12.8-47.6 21.4-22.5 19.2-23.4 51.8-1.9 70.8C33 342.4 45.2 348.4 56 351c16.8 4 39.8 2.2 56-4.4 25.3-10.3 40.7-32.8 36-52.7-3-12.9-9.1-22.3-18.8-29.1-2.4-1.6-4.1-3.5-4-4 .7-2 13.1-8.3 37.8-19.2 13.5-5.9 34.1-15.7 45.8-21.6l21.3-10.9 70.2 39.8c38.6 21.8 78 44.1 87.5 49.5 28.3 16 39.2 19.1 64.5 18.3 11-.3 18.3-1 22.7-2.2 8.6-2.3 23.2-8.5 22.8-9.7-.2-.5-48.4-29.2-107.1-63.6C332 206.7 284 178.3 284 178s46-27.6 102.3-60.7c56.2-33.2 105-61.9 108.5-64 3.4-2 6.2-4 6.2-4.3 0-.7-10.2-5.9-15.5-7.8-19-7-42.7-6.5-66.6 1.4-12.2 4.1-15.4 5.8-127 68.7L230.4 146l-24.5-12.3c-13.4-6.7-33.4-16.2-44.4-21.1-21.9-9.7-30-13.6-35.7-17.2l-3.7-2.4 7.9-7.8c15.7-15.5 19.7-33 11.5-49.8C134.2 20.5 117.6 8 98.4 2.9 87.7.1 64.5-.6 54.5 1.6M81.2 24c9.8 1.4 17 4.5 22.9 9.8 17.2 15.4 11.4 41.3-10.9 48.2-7 2.2-17 2.6-24.1.9-11.6-2.6-23.9-11.7-28.9-21.4-6.6-12.7-.7-27.6 13.5-34.2 8.8-4.1 16.1-4.9 27.5-3.3m17 247.5c6.3 2.2 13.6 8.8 16.4 15 2.7 5.7 3.2 14.2 1.2 20.8-1.5 5.1-9.2 13.5-15.6 17-10.6 5.8-29.1 7.3-40.1 3.3-10.6-3.9-19.1-13.8-19.9-23.1-1.1-13.7 12.8-28.9 31.1-34 6.5-1.9 20.3-1.3 26.9 1"/>
                        </svg>
                    </div>
                    <div class="square-button" :class="{ 'active-square-button': loopEnabled }" @click="loopEnabled = !loopEnabled"  title="Loop">
                        <svg width="20px" height="20px" viewBox="-24 0 512 512" xmlns="http://www.w3.org/2000/svg" >
                            <path d="M232 448Q186 448 148 425 109 402 87 363 64 324 64 278L64 256 112 256 112 280Q112 331 147 366 182 400 233 400 265 400 293 384 320 367 336 340 352 313 352 281 352 230 318 195 283 160 232 160L232 232 136 136 232 40 232 112Q279 112 317 134 355 157 378 196 400 234 400 280 400 327 378 364 356 403 317 426 277 448 232 448Z" />
                        </svg>
                    </div>                
                </div>
                <div class="volume-wrap">
                    <span>{{ volumeDisplay }}</span>
                    <input type="range" class="volume-control" name="volume" min="0" max="2" step="0.01" v-model="audioVolume" @input="changeVolume"/>
                </div>
            </div>
        </div>
        <div ref="ContainerRef" class="audio-visual" @click="seek" @mousemove="hoverSeek" @mouseleave="hoverLeave">
            <label v-show="showUpload" class="add-audio-label"><div class="circle">+</div>
                <input type="file" accept="audio/*" @change="handleFile" hidden />
            </label>
            <audio ref="AudioRef" accept="audio/*" hidden @play="startPlay" @pause="pausePlay" @loadedmetadata="setDuration"></audio>

            <div class="trim-overlay left" :style="{ width: startPercent + '%' }" v-if="fileAdded"></div>
            <div class="trim-handle start" :style="{ left: startPercent + '%' }" draggable="false" @mousedown.stop="startDrag('start')" @mouseenter="hoveringHandle = true" @mouseleave="hoveringHandle = false" v-if="fileAdded">
                <div class="trim-timer">{{formatTime(trimStart)}}</div>
            </div>

            <canvas ref="CanvasRef" class="canvas"></canvas>

            <div v-show="showHoverBar" class="progress-bar hover-bar" :style="{ left: hoverSeekX + 'px' }">
                <div class="trim-timer">{{formatTime(hoverSeekTime)}}</div>
            </div>
            <div class="progress-bar" v-if="fileAdded" :style="{ left: playProgress + 'px' }"></div>

            <div class="trim-handle end" :style="{ left: endPercent + '%' }" draggable="false" @mousedown.stop="startDrag('end')" @mouseenter="hoveringHandle = true" @mouseleave="hoveringHandle = false" v-if="fileAdded">
                <div class="trim-timer">{{formatTime(trimEnd)}}</div>
            </div>
            <div class="trim-overlay right" :style="{ width: (100 - endPercent) + '%' }" v-if="fileAdded"></div>
        </div>
    </div>
</template>

<style scoped>

.channel
{
    position: relative;
    border: 1px solid #0da90d;
    background-color: white;
}

.control-bar
{
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 10px 0px 30px 0px;
    border-bottom: 2px solid #e9e9e9;
    color: #ff3b3b;
    font-weight: bold;
}

.left-wrapper, .right-wrapper 
{
    padding: 0px 20px;
    flex: 1;
    display: flex;
    align-items: center;
}

.right-wrapper 
{
    justify-content: flex-end;
}

.center-wrapper 
{
    display: flex;
    justify-content: center;
    align-items: center;
    flex-direction: column;
    gap: 10px;
}

.audio-visual
{
    position: relative;
    display: flex;
    justify-content: center;
    height: 86px;
    margin: 0px 30px;
}

.add-audio-label
{
    font-size: 30px;
    font-weight: bold;
    cursor: pointer;
    z-index: 10;
    display: flex;
    justify-content: center;
    flex-direction: column;
}

.circle
{
    width: 40px;
    height: 40px;
    border: 2px solid #0da90d;
    border-radius: 30px;
    text-align: center;
    line-height: 34px;
    transition: all .2s ease;
}

.circle:hover
{
    background-color: #0da90d;
    color: white;
}

.play-audio-wrapper 
{
  width: 50px;
  height: 50px;
  border-radius: 50%;
  border: none;
  cursor: pointer;

  display: flex;
  align-items: center;
  justify-content: center;

  background: linear-gradient(135deg, #ff6a6a, #ff3b3b);
  box-shadow:
    0 4px 10px rgba(0, 0, 0, 0.25),
    inset 0 1px 0 rgba(255, 255, 255, 0.15);
  transition:
    transform 0.12s ease,
    box-shadow 0.12s ease,
    filter 0.12s ease;
}

.play-audio-wrapper svg 
{
  width: 20px;
  height: 20px;
  fill: white;
}

.play-audio-wrapper svg.play
{
    padding-left: 4px;
}

.play-audio-wrapper:hover 
{
  transform: scale(1.02);
  filter: brightness(1.05);
}

.play-audio-wrapper:active 
{
  transform: translateY(0);
  box-shadow:
    0 2px 6px rgba(0, 0, 0, 0.25),
    inset 0 2px 4px rgba(0, 0, 0, 0.25);
}

canvas
{
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
}

.progress-bar 
{
  position: absolute;
  top: 0;
  z-index: 10;
  width: 2px;
  height: 100%;
  pointer-events: none;

  background: linear-gradient(
    to bottom,
    rgba(255, 80, 80, 0.95),
    rgba(255, 80, 80, 0.45)
  );

  box-shadow: 0 0 6px rgba(255, 80, 80, 0.45);
}

.hover-bar 
{
  background: linear-gradient(
    to bottom,
    rgba(255, 80, 80, 0.55),
    rgba(255, 80, 80, 0.25)
  );

  box-shadow: 0 0 4px rgba(255, 80, 80, 0.25);
}

.hover-bar::before 
{
  background: rgba(255, 80, 80, 0.7);
  box-shadow:
    0 0 0 2px rgba(255, 80, 80, 0.15),
    0 0 6px rgba(255, 80, 80, 0.35);
}

.trim-handle 
{
  position: absolute;
  top: 0;
  width: 14px;
  height: 100%;
  transform: translateX(-50%);
  z-index: 20;
  touch-action: none;

  cursor: ew-resize;
  display: flex;
  align-items: center;
  justify-content: center;

  background: rgb(0, 120, 212);
  border-radius: 4px;
}

.trim-handle::before 
{
  content: "";
  width: 3px;
  height: 40%;
  border-radius: 2px;
  background: rgba(171, 203, 255, 0.9);
}

.trim-timer
{
    position: absolute;
    bottom: 100%;
    left: 50%;
    transform: translateX(-50%);
    margin-bottom: 6px;
    padding: 4px 8px;
    font-size: 12px;
    font-family: sans-serif;
    border-radius: 4px;
    white-space: nowrap;
    box-shadow: 0 2px 6px rgba(0,0,0,0.3);
    user-select: none;
    background: white;
}

.trim-timer::after 
{
  content: '';
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  border-width: 5px;
  border-style: solid;
  border-color: white transparent transparent transparent;
}

.trim-overlay
{
    position: absolute;
    height: 100%;
    top: 0;
    opacity: .3;
    background-color: black;
    z-index: 9;
}

.left
{
    left: 0;
}

.right
{
    right: 0;
}

.button-wrap
{
    display: flex;
    margin: 0px 50px;
}

.square-button
{
    height: 20px;
    width: 20px;
    border-radius: 4px;
    padding: 5px;
    transition: all .15s ease;
    cursor: pointer;
}

.square-button:hover {
  background: #e9e9e9;
  box-shadow:
    inset 0 1px 2px rgba(0, 0, 0, 0.6),
    inset 0 -1px 1px rgba(255, 255, 255, 0.05);
}

.active-square-button
{
    background-color: #d8d8d8;
    box-shadow:
    inset 0 1px 2px rgba(0, 0, 0, 0.6),
    inset 0 -1px 1px rgba(255, 255, 255, 0.05);
}

.audio-info
{
    font-size: 12px;
}

.volume-wrap
{
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

</style>