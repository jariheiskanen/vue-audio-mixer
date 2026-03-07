<script setup>
import { ref, watch, onMounted } from 'vue';

const props = defineProps({
  modelValue: { type: Number, required: true }, //trim timer
  precision: { type: Number, default: 2 } //timer precision
})

//value of trim input
const emit = defineEmits(['update:modelValue']);

const inputValue = ref('');
const isEditing = ref(false); 

//input in focus
function startEditing() 
{
  isEditing.value = true;
  inputValue.value = formatTime(props.modelValue, props.precision);
}

//update value and send manually inputted value to AudioChannel
function commitEdit() 
{
  isEditing.value = false;
  const parsed = timeToSeconds(inputValue.value);
  if(parsed !== null) 
  {
    emit('update:modelValue', parsed);
  }
  inputValue.value = formatTime(parsed ?? props.modelValue, props.precision);
}

//commit on enter, up and down to change timer
function handleKeydown(e) 
{
  if(e.key === 'Enter')
  {
    commitEdit();
  }
  if (e.key === "ArrowUp" || e.key === "ArrowDown") 
  {
    e.preventDefault();

    const step = Math.pow(10, -props.precision); // 0.01 if precision=2
    let value = timeToSeconds(inputValue.value);

    if (e.key === "ArrowUp") value += step;
    if (e.key === "ArrowDown") value -= step;

    inputValue.value = formatTime(value, props.precision);
    commitEdit();
  }
}

// Format seconds to mm:ss.M or mm:ss.MM
function formatTime(sec, precision = 1) {
    sec = Number(sec);

    if (sec >= 0) 
    {
        const minutes = Math.floor(sec / 60);
        const seconds = Math.floor(sec % 60);

        const fractionMultiplier = Math.pow(10, precision);
        const fraction = Math.round((sec % 1) * fractionMultiplier).toString().padStart(precision, '0');

        return `${minutes}:${seconds.toString().padStart(2, '0')}.${fraction}`;
    } 
    else 
    {
        return precision === 2 ? "0:00.00" : "0:00.0";
    }
}

//converts mm:ss:MM into ss:MM
function timeToSeconds(timeString) 
{
  const match = timeString.match(/^(\d+):(\d{2})\.(\d{1,2})$/);
  if (!match) return null;

  const minutes = Number(match[1]);
  const seconds = Number(match[2]);
  const fraction = Number(match[3]) / Math.pow(10, match[3].length);

  return minutes * 60 + seconds + fraction;
}

// Sync when external value changes (but not while editing)
watch(
  () => props.modelValue,
  (newVal) => {
    if (!isEditing.value) inputValue.value = formatTime(newVal, props.precision)
  }
)

onMounted(() => {
  inputValue.value = formatTime(props.modelValue, props.precision)
})
</script>

<template>
  <div class="trim-timer" @mousedown.stop @click.stop>
    <input class="trim-timer-input" v-model="inputValue" @focus="startEditing" @blur="commitEdit" @keydown="handleKeydown"/>
  </div>
</template>

<style scoped>
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
      cursor: auto;
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
  .trim-timer-input
  {
      max-width: 40px;
      background: transparent;
      border: none;
      border-bottom: 1px solid currentColor;
      color: inherit;
      text-align: center;
      font-size: 12px;
      padding: 0 2px;
      outline: none;
      background-color: #fff4f0;
      transition: border .2s linear;
      border-bottom-color: #ff3b3b;
  }

  .trim-timer-input:hover, .trim-timer-input:focus
  {
      border-bottom-color: #ff6b6b;
      background-color: #fff0ea;
  }
</style>
