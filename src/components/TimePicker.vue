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

function scrollTo(container: HTMLElement | null, value: number) {
  if (!container) return;
  container.scrollTop = value * itemHeight;
}

function handleScroll(type: 'hour' | 'minute', e: Event) {
  if (isDragging) return; // Don't snap while dragging (handled on mouseup/leave)

  const target = e.target as HTMLElement;
  snap(target, type);
}

function snap(target: HTMLElement, type: 'hour' | 'minute') {
  const index = Math.round(target.scrollTop / itemHeight);
  if (type === 'hour') {
    const val = hours[index];
    if (val !== undefined && val !== props.hour) emit('update:hour', val);
  } else {
    const val = minutes[index];
    if (val !== undefined && val !== props.minute) emit('update:minute', val);
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
  snap(currentContainer, type);
  // Manually smooth scroll to snap point if needed (CSS snap handles it mostly, but explicit set helps)
  const index = Math.round(currentContainer.scrollTop / itemHeight);
  currentContainer.scrollTo({ top: index * itemHeight, behavior: 'smooth' });

  currentContainer = null;
  document.removeEventListener('mousemove', onMouseMove);
  document.removeEventListener('mouseup', onMouseUp);
}

// Initial scroll position
onMounted(() => {
  nextTick(() => {
    scrollTo(hourContainer.value, props.hour);
    scrollTo(minuteContainer.value, props.minute);
  });
});

// Watch for external changes
watch(() => props.hour, (newVal) => {
  if (hourContainer.value && !isDragging && Math.round(hourContainer.value.scrollTop / itemHeight) !== newVal) {
    scrollTo(hourContainer.value, newVal);
  }
});

watch(() => props.minute, (newVal) => {
  if (minuteContainer.value && !isDragging && Math.round(minuteContainer.value.scrollTop / itemHeight) !== newVal) {
    scrollTo(minuteContainer.value, newVal);
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
      <div class="spacer"></div>
      <div v-for="h in hours" :key="h" class="item" :class="{ active: h === hour }">
        {{ pad(h) }}
      </div>
      <div class="spacer"></div>
    </div>

    <div class="colon">:</div>

    <!-- Minutes -->
    <div 
      class="column" 
      ref="minuteContainer" 
      @scroll.passive="handleScroll('minute', $event)"
      @mousedown="onMouseDown($event, minuteContainer)"
    >
      <div class="spacer"></div>
      <div v-for="m in minutes" :key="m" class="item" :class="{ active: m === minute }">
        {{ pad(m) }}
      </div>
      <div class="spacer"></div>
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

.spacer {
  height: 80px; /* (200 - 40) / 2 */
}

.colon {
  font-size: 24px;
  font-weight: 600;
  margin: 0 10px;
  padding-bottom: 4px;
  color: #000;
}
</style>
