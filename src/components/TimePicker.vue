<script setup lang="ts">
import { ref, onMounted, watch, nextTick } from 'vue';

const props = defineProps<{
  hour: number;
  minute: number;
}>();

const emit = defineEmits<{
  (e: 'update:hour', val: number): void;
  (e: 'update:minute', val: number): void;
}>();

const hours = Array.from({ length: 24 }, (_, i) => i);
const minutes = Array.from({ length: 60 }, (_, i) => i);

// Triple the lists for infinite scroll illusion
const displayHours = [...hours, ...hours, ...hours];
const displayMinutes = [...minutes, ...minutes, ...minutes];

const hourContainer = ref<HTMLElement | null>(null);
const minuteContainer = ref<HTMLElement | null>(null);
const itemHeight = 40; // matches css height

// Mouse Drag State
let isDragging = false;
let startY = 0;
let startScrollTop = 0;
let currentContainer: HTMLElement | null = null;

function pad(n: number) {
  return n.toString().padStart(2, '0');
}

function scrollTo(container: HTMLElement | null, value: number, type: 'hour' | 'minute') {
  if (!container) return;
  // Start in the middle set
  const offset = type === 'hour' ? 24 : 60;
  const index = offset + value;
  container.scrollTop = index * itemHeight - 80; // 80 = (200/2) - (40/2)
}

function handleScroll(type: 'hour' | 'minute', e: Event) {
  if (isDragging) return; 

  const target = e.target as HTMLElement;
  checkLoop(target, type);
  snap(target, type);
}

function checkLoop(target: HTMLElement, type: 'hour' | 'minute') {
  const count = type === 'hour' ? 24 : 60;
  const singleSetHeight = count * itemHeight;
  const centerIndex = Math.round((target.scrollTop + 80) / itemHeight);
  
  // If we are in the first set (too high), jump to middle
  if (centerIndex < count) {
    target.scrollTop += singleSetHeight;
  } 
  // If we are in the third set (too low), jump to middle
  else if (centerIndex >= count * 2) {
    target.scrollTop -= singleSetHeight;
  }
}

function snap(target: HTMLElement, type: 'hour' | 'minute') {
  const centerIndex = Math.round((target.scrollTop + 80) / itemHeight);
  // Safe access using modulo or just relying on the array logic
  // Since we jump, centerIndex should be roughly in [count, 2*count]
  // But strictly, we can just look up the value
  const list = type === 'hour' ? displayHours : displayMinutes;
  const val = list[centerIndex];
  
  if (val !== undefined) {
    if (type === 'hour') {
      if (val !== props.hour) emit('update:hour', val);
    } else {
      if (val !== props.minute) emit('update:minute', val);
    }
  }
}

// Mouse Event Handlers
function onMouseDown(e: MouseEvent, container: HTMLElement | null) {
  if (!container) return;
  isDragging = true;
  startY = e.clientY;
  startScrollTop = container.scrollTop;
  currentContainer = container;
  container.style.scrollSnapType = 'none'; // Disable snap while dragging
  document.addEventListener('mousemove', onMouseMove);
  document.addEventListener('mouseup', onMouseUp);
}

function onMouseMove(e: MouseEvent) {
  if (!isDragging || !currentContainer) return;
  e.preventDefault();
  const deltaY = startY - e.clientY;
  currentContainer.scrollTop = startScrollTop + deltaY;
}

function onMouseUp() {
  if (!isDragging || !currentContainer) return;
  isDragging = false;
  
  // Re-enable snap and force snap to nearest
  currentContainer.style.scrollSnapType = 'y mandatory';
  
  // Identify type based on container ref
  const type = currentContainer === hourContainer.value ? 'hour' : 'minute';
  
  checkLoop(currentContainer, type);
  snap(currentContainer, type);
  
  // Manually smooth scroll to snap point
  const centerIndex = Math.round((currentContainer.scrollTop + 80) / itemHeight);
  currentContainer.scrollTo({ top: centerIndex * itemHeight - 80, behavior: 'smooth' });

  currentContainer = null;
  document.removeEventListener('mousemove', onMouseMove);
  document.removeEventListener('mouseup', onMouseUp);
}

// Initial scroll position
onMounted(() => {
  nextTick(() => {
    scrollTo(hourContainer.value, props.hour, 'hour');
    scrollTo(minuteContainer.value, props.minute, 'minute');
  });
});

// Watch for external changes
watch(() => props.hour, (newVal) => {
  if (hourContainer.value && !isDragging) {
    const centerIndex = Math.round((hourContainer.value.scrollTop + 80) / itemHeight);
    if (displayHours[centerIndex] !== newVal) {
       scrollTo(hourContainer.value, newVal, 'hour');
    }
  }
});

watch(() => props.minute, (newVal) => {
  if (minuteContainer.value && !isDragging) {
    const centerIndex = Math.round((minuteContainer.value.scrollTop + 80) / itemHeight);
    if (displayMinutes[centerIndex] !== newVal) {
      scrollTo(minuteContainer.value, newVal, 'minute');
    }
  }
});
</script>

<template>
  <div class="time-picker">
    <!-- Highlight bar -->
    <div class="highlight"></div>

    <!-- Hours -->
    <div 
      class="column" 
      ref="hourContainer" 
      @scroll.passive="handleScroll('hour', $event)"
      @mousedown="onMouseDown($event, hourContainer)"
    >
      <div v-for="(h, i) in displayHours" :key="i" class="item" :class="{ active: h === hour }">
        {{ pad(h) }}
      </div>
    </div>

    <div class="colon">:</div>

    <!-- Minutes -->
    <div 
      class="column" 
      ref="minuteContainer" 
      @scroll.passive="handleScroll('minute', $event)"
      @mousedown="onMouseDown($event, minuteContainer)"
    >
      <div v-for="(m, i) in displayMinutes" :key="i" class="item" :class="{ active: m === minute }">
        {{ pad(m) }}
      </div>
    </div>
  </div>
</template>

<style scoped>
.time-picker {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 200px;
  overflow: hidden;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  font-variant-numeric: tabular-nums;
  background: #f5f5f7;
  border-radius: 12px;
  user-select: none; /* Prevent text selection while dragging */
}

.highlight {
  position: absolute;
  top: 50%;
  left: 10px;
  right: 10px;
  height: 40px;
  margin-top: -20px;
  background: rgba(0, 0, 0, 0.05);
  border-radius: 8px;
  pointer-events: none;
}

.column {
  height: 100%;
  width: 60px;
  overflow-y: auto;
  scroll-snap-type: y mandatory;
  scrollbar-width: none; /* Firefox */
  z-index: 1;
  cursor: grab;
}

.column:active {
  cursor: grabbing;
}

.column::-webkit-scrollbar {
  display: none;
}

.item {
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  scroll-snap-align: center;
  font-size: 24px;
  color: #8e8e93;
  transition: all 0.2s;
}

.item.active {
  color: #000;
  font-weight: 600;
  transform: scale(1.1);
}

.colon {
  font-size: 24px;
  font-weight: 600;
  margin: 0 10px;
  padding-bottom: 4px;
  color: #000;
}
</style>
