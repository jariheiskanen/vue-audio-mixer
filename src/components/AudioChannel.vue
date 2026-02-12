<script setup>
import { ref } from 'vue';

const AudioRef = ref(null);
const CanvasRef = ref(null);
const ContainerRef = ref(null);

const showUpload = ref(true); //shows upload button when audio file hasn't been added
const audioPlaying = ref(false); //true when audio file is being played
const fileAdded = ref(false); //file added
const audioCtx = new AudioContext();
const playProgress = ref(0); //progress of the audio file
const audioDuration = ref(0); //duration of audio file
const audioCurrent = ref(0); //current time of audio

let animationId = null; //used for audio progress animation
let animationId2 = null; //used for timer progress

//process uploaded file
async function decodeFile(file) 
{
  const arrayBuffer = await file.arrayBuffer();
  return await audioCtx.decodeAudioData(arrayBuffer);
}

//get volume data for file at set sample rate
function getVolumeData(audioBuffer, samples = 500) 
{
    const channelData = audioBuffer.getChannelData(0); // mono
    const blockSize = Math.floor(channelData.length / samples);
    const volumes = [];

    for (let i=0; i<samples; i++) 
    {
        let sum = 0;
        const start = i*blockSize;

        for (let j=0; j<blockSize; j++) 
        {
            const sample = channelData[start + j];
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
}

//file upload
async function handleFile(e) 
{
    const file = e.target.files[0];
    if (!file) return;

    //hide upload icon
    showUpload.value = false;
    //set fileAdded ref
    fileAdded.value = true;

    const fileURL = URL.createObjectURL(file);
    AudioRef.value.src = fileURL;
    AudioRef.value.load();

    //process, normalize and smooth value before drawing it to canvas
    const audioBuffer = await decodeFile(file);
    const volumeData = getVolumeData(audioBuffer, 600);
    const normalized = normalize(volumeData);
    const smoothed = smooth(normalized);

    drawVolumeGraph(CanvasRef.value, smoothed);
}

//change volume
function changeVolume(e)
{
    AudioRef.value.volume = e.target.value;
}

//plays/pauses audio
function togglePlay()
{
    if(!audioPlaying.value)
    {
        AudioRef.value.play();
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

  playProgress.value = (audio.currentTime / audio.duration) * container.clientWidth;

  animationId = requestAnimationFrame(animate);
}

//updates current time of audio file
function updateTime() 
{
  if (!AudioRef.value) return;
  audioCurrent.value = AudioRef.value.currentTime;
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

//runs when audio file has finished playing
function endPlay()
{
    cancelAnimationFrame(animationId);
    cancelAnimationFrame(animationId2);
    audioPlaying.value = false;
    audioCurrent.value = 0;
    playProgress.value = 0;
}

//skip through audio by clicking
function seek(e) {
    if (!fileAdded.value) return;
    const audio = AudioRef.value;
    const container = ContainerRef.value;
    const rect = container.getBoundingClientRect();

    const clickX = e.clientX - rect.left;
    const percent = clickX / rect.width;

    audio.currentTime = percent * audio.duration;
    playProgress.value = (audio.currentTime / audio.duration) * container.clientWidth;

    audioCurrent.value = audio.currentTime;
}

//sets total length of audio
function setDuration(e)
{
    audioDuration.value = e.target.duration;
}

//Format seconds to mm:ss.M
function formatTime(sec) {
    sec = Number(sec);
    const minutes = Math.floor(sec / 60);
    const seconds = Math.floor(sec % 60);
    const tenths = Math.floor((sec % 1) * 10);
    return `${minutes}:${seconds.toString().padStart(2,'0')}.${tenths}`;
}

</script>

<template>
    <div class="channel">
        <div class="control-bar" v-if="fileAdded">
            <div class="play-audio-wrapper" @click="togglePlay">
                <svg v-if="!audioPlaying" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 50 50" width="20" height="20">
                    <path d="M5.716 49.715a2.272 2.272 0 0 0 2.306 -0.061l36.364 -22.727a2.273 2.273 0 0 0 0 -3.855L8.023 0.345A2.273 2.273 0 0 0 4.545 2.273v45.455a2.273 2.273 0 0 0 1.171 1.988"/>
                </svg>
                <svg v-else width="20px" height="20px" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg">
                    <path d="M4.694 1.758A0.787 0.787 0 0 0 3.906 2.545v14.909A0.787 0.787 0 0 0 4.694 18.242h2.566A0.787 0.787 0 0 0 8.047 17.455V2.545A0.787 0.787 0 0 0 7.259 1.758zm8.047 0A0.787 0.787 0 0 0 11.953 2.545v14.909A0.787 0.787 0 0 0 12.741 18.242h2.566A0.787 0.787 0 0 0 16.094 17.455V2.545A0.787 0.787 0 0 0 15.306 1.758z"/>
                </svg>
            </div>

            <span>{{formatTime(audioCurrent) +" / "}}</span>
            <span>{{formatTime(audioDuration)}}</span>

            <div><input type="range" class="volume-control" name="volume" min="0" max="1" step="0.01" @input="changeVolume"/></div>
        </div>
        <div ref="ContainerRef" class="audio-visual" @click="seek">
            <label v-show="showUpload" class="add-audio-label"><div class="circle">+</div>
                <input type="file" accept="audio/*" @change="handleFile" hidden />
            </label>
            <audio ref="AudioRef" accept="audio/*" hidden @play="startPlay" @pause="pausePlay" @ended="endPlay" @loadedmetadata="setDuration"></audio>

            <canvas ref="CanvasRef" class="canvas"></canvas>
            <div class="progress-bar" v-if="fileAdded" :style="{ left: playProgress + 'px' }"></div>
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
    padding: 10px 0px;
}

.audio-visual
{
    position: relative;
    display: flex;
    justify-content: center;
    height: 86px;
    background-color: #cbcbcb;
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
    line-height: 40px;
    transition: all .2s ease;
}

.circle:hover
{
    background-color: #0da90d;
    color: white;
}

.play-audio-wrapper
{
    padding: 0px 10px;
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
  background: red;
  pointer-events: none;
}

</style>