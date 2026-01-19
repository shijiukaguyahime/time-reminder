<script setup lang="ts">
import { Check } from 'lucide-vue-next';

// --- Types ---
type RepeatType = 'once' | 'daily' | 'workdays' | 'mon_sat' | 'custom' | 'shift';

interface Alarm {
  id: string;
  hour: number;
  minute: number;
  label: string;
  enabled: boolean;
  repeat: {
    type: RepeatType;
    customDays: number[];
    shift?: {
      startDate: string;
      workDays: number;
      restDays: number;
    };
  };
}

const props = defineProps<{
  alarm: Alarm;
  isSelectionMode: boolean;
  isSelected: boolean;
}>();

const emit = defineEmits<{
  (e: 'toggle', id: string, enabled: boolean): void;
  (e: 'delete', id: string): void;
  (e: 'edit', id: string): void;
  (e: 'longpress', id: string): void;
  (e: 'select', id: string): void;
}>();

// --- Logic ---
function pad(n: number) {
  return n.toString().padStart(2, '0');
}

function getRepeatText(repeat: Alarm['repeat']) {
  const { type, customDays, shift } = repeat;
  const map: Record<string, string> = {
    once: '仅一次',
    daily: '每天',
    workdays: '工作日 (周一至周五)',
    mon_sat: '工作日 (周一至周六)',
    custom: '自定义',
    shift: '轮班',
  };
  
  if (type === 'custom') {
    const days = ['周日', '周一', '周二', '周三', '周四', '周五', '周六'];
    if (customDays.length === 0) return '自定义';
    return customDays.sort().map(d => days[d]).join(' ');
  }
  
  if (type === 'shift' && shift) {
    return `轮班 (${shift.workDays}班${shift.restDays}休)`;
  }
  
  return map[type] || type;
}

// --- Long Press Only ---
let startX = 0;
let startY = 0;
let longPressTimer: any = null;
let longPressTriggered = false;

function onTouchStart(e: TouchEvent | MouseEvent) {
  if (props.isSelectionMode) return;
  
  const clientX = 'touches' in e ? e.touches[0].clientX : e.clientX;
  const clientY = 'touches' in e ? e.touches[0].clientY : e.clientY;
  
  startX = clientX;
  startY = clientY;
  longPressTriggered = false;
  
  // Long press detection
  longPressTimer = setTimeout(() => {
    longPressTriggered = true;
    emit('longpress', props.alarm.id);
  }, 500);
}

function onTouchMove(e: TouchEvent | MouseEvent) {
  if (props.isSelectionMode) return;
  
  const clientX = 'touches' in e ? e.touches[0].clientX : e.clientX;
  const clientY = 'touches' in e ? e.touches[0].clientY : e.clientY;
  
  const deltaX = clientX - startX;
  const deltaY = clientY - startY;
  
  // Cancel long press if moved significantly
  if (Math.abs(deltaX) > 10 || Math.abs(deltaY) > 10) {
    clearTimeout(longPressTimer);
  }
}

function onTouchEnd() {
  clearTimeout(longPressTimer);
}

// Mouse compat
function onMouseDown(e: MouseEvent) {
  onTouchStart(e);
  document.addEventListener('mousemove', onMouseMove);
  document.addEventListener('mouseup', onMouseUp);
}

function onMouseMove(e: MouseEvent) {
  onTouchMove(e);
}

function onMouseUp() {
  onTouchEnd();
  document.removeEventListener('mousemove', onMouseMove);
  document.removeEventListener('mouseup', onMouseUp);
}

function handleClick() {
  if (longPressTriggered) {
    longPressTriggered = false;
    return;
  }

  if (props.isSelectionMode) {
    emit('select', props.alarm.id);
  } else {
    emit('edit', props.alarm.id);
  }
}
</script>

<template>
  <div class="alarm-item-container" :class="{ 'selection-mode': isSelectionMode }">
    <!-- Main Content -->
    <div 
      class="alarm-content"
      @touchstart="onTouchStart"
      @touchmove="onTouchMove"
      @touchend="onTouchEnd"
      @mousedown="onMouseDown"
      @click.stop="handleClick"
    >
      <!-- Selection Checkbox -->
      <div v-if="isSelectionMode" class="checkbox" :class="{ checked: isSelected }">
        <Check v-if="isSelected" :size="14" color="white" />
      </div>

      <div class="info">
        <div class="time">
          {{ pad(alarm.hour) }}:{{ pad(alarm.minute) }}
        </div>
        <div class="details">
          <span class="label" v-if="alarm.label">{{ alarm.label }}，</span>
          <span class="repeat">{{ getRepeatText(alarm.repeat) }}</span>
        </div>
      </div>
      
      <div class="actions" @click.stop v-if="!isSelectionMode">
        <label class="switch">
          <input 
            type="checkbox" 
            :checked="alarm.enabled" 
            @change="emit('toggle', alarm.id, ($event.target as HTMLInputElement).checked)"
          >
          <span class="slider round"></span>
        </label>
      </div>
    </div>
  </div>
</template>

<style scoped>
.alarm-item-container {
  position: relative;
  margin-bottom: 12px;
  height: 80px; /* Fixed height for simplicity */
  border-radius: 12px;
  overflow: hidden;
  background: white;
}

.alarm-item-container.selection-mode {
  background: transparent;
}

.alarm-content {
  position: relative;
  width: 100%;
  height: 100%;
  background: white;
  display: flex;
  align-items: center;
  padding: 0 20px 0 16px; /* Increased right padding to prevent switch overflow */
  cursor: pointer;
  box-sizing: border-box;
}

.alarm-content:active {
  background: #f9f9f9;
}

.checkbox {
  width: 22px;
  height: 22px;
  border-radius: 50%;
  border: 2px solid #c7c7cc;
  margin-right: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: all 0.2s;
}

.checkbox.checked {
  background: #007aff;
  border-color: #007aff;
}

.info {
  flex: 1;
  min-width: 0; /* Allow text truncation if needed */
  margin-right: 16px; /* Space between text and switch */
}

.actions {
  flex-shrink: 0; /* Prevent switch from shrinking */
  display: flex;
  align-items: center;
}

.time {
  font-size: 32px;
  font-weight: 300;
  color: #000;
  line-height: 1;
}

.details {
  margin-top: 4px;
  font-size: 13px;
  color: #8e8e93;
}

/* Switch Styles */
.switch {
  position: relative;
  display: inline-block;
  width: 50px;
  height: 30px;
}

.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: #ccc;
  transition: .4s;
}

.slider:before {
  position: absolute;
  content: "";
  height: 26px;
  width: 26px;
  left: 2px;
  bottom: 2px;
  background-color: white;
  transition: .4s;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

input:checked + .slider {
  background-color: #34c759;
}

input:checked + .slider:before {
  transform: translateX(20px);
}

.slider.round {
  border-radius: 34px;
}

.slider.round:before {
  border-radius: 50%;
}
</style>
