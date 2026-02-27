<script setup>
import {computed} from 'vue';

const props = defineProps({
  modelValue: {
    type: Number,
    required: true
  }
});

const emit = defineEmits(['update:modelValue']);

// Compute current volume display
const volumeDisplay = computed(() => Math.round(props.modelValue * 100) + "%");

//emit value on input change
const handleInput = (event) => {
  emit('update:modelValue', Number(event.target.value));
};
</script>

<template>
  <div class="volume-wrap">
    <span>{{ volumeDisplay }}</span>
    <input type="range" class="volume-control" name="volume" min="0" max="2" step="0.01" :value="modelValue" @input="handleInput"/>
  </div>
</template>

<style scoped>
.volume-wrap
{
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}
</style>
