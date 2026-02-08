<script setup>
import { ref } from 'vue';

const CanvasRef = ref(null);
const showUpload = ref(true);
const audioCtx = new AudioContext();

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

    //process, normalize and smooth value before drawing it to canvas
    const audioBuffer = await decodeFile(file);
    const volumeData = getVolumeData(audioBuffer, 600);
    const normalized = normalize(volumeData);
    const smoothed = smooth(normalized);

    drawVolumeGraph(CanvasRef.value, smoothed);
}

</script>

<template>
    <div class="channel">
        <label v-show="showUpload" class="add-audio-label"><div class="circle">+</div>
            <input type="file" accept="audio/*" @change="handleFile" hidden />
        </label>
        <canvas ref="CanvasRef" class="canvas"></canvas>
    </div>
</template>

<style scoped>

.channel
{
    position: relative;
    border: 1px solid #0da90d;
    display: flex;
    justify-content: center;
    background-color: #cbcbcb;
    height: 86px;
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

canvas
{
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
}

</style>