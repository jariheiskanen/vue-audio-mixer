<script setup>
import { ref } from 'vue';

const AudioRef = ref(null);

function handleFile(e) 
{
    const file = e.target.files[0];
    const url = URL.createObjectURL(file);

    //load selected file into audio element
    AudioRef.value.src = url;
    AudioRef.value.load();

    //analyze audio with Web Audio API
    //visualize on canvas
}

</script>

<template>
    <div class="channel">
        <label class="add-audio-label"><div class="circle">+</div>
            <input type="file" accept="audio/*" @change="handleFile" hidden />
        </label>

        <audio ref="AudioRef" class="audio" hidden></audio>
        <canvas class="canvas"></canvas>
    </div>
</template>

<style scoped>

.channel
{
    position: relative;
    border: 1px solid #0da90d;
    padding: 20px;
    display: flex;
    justify-content: center;
    background-color: #cbcbcb;
}

.add-audio-label
{
    font-size: 30px;
    font-weight: bold;
    cursor: pointer;
}

.circle
{
    width: 40px;
    height: 40px;
    border: 2px solid #0da90d;
    border-radius: 30px;
    text-align: center;
    line-height: 40px;
}

canvas
{
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

</style>