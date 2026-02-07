<!--
npm run dev
npm run build
npm run deploy
-->

<script setup>
import AudioChannel from './components/AudioChannel.vue'
</script>

<template>
  <div class="layout-wrapper">
    <div class="main-section">
      <!--main section here-->
    </div>
    <div class="channel-wrapper">
      <AudioChannel />
      <AudioChannel />
      <AudioChannel />
      <AudioChannel />
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
}
</style>
