<script setup lang="ts">
import { ref } from 'vue';
import { ChevronDown, Check } from 'lucide-vue-next';

interface Option {
  label: string;
  value: string;
}

const props = defineProps<{
  modelValue: string;
  options: Option[];
}>();

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void;
}>();

const isOpen = ref(false);

function select(value: string) {
  emit('update:modelValue', value);
  isOpen.value = false;
}

function toggle() {
  isOpen.value = !isOpen.value;
}

// Close when clicking outside (simple implementation using v-show for now)
// In a full app, use click-outside directive
</script>

<template>
  <div class="custom-select" :class="{ open: isOpen }">
    <div class="trigger" @click="toggle">
      <span class="label">{{ options.find(o => o.value === modelValue)?.label || modelValue }}</span>
      <ChevronDown class="icon" :size="16" />
    </div>
    
    <transition name="dropdown">
      <div v-if="isOpen" class="dropdown">
        <div 
          v-for="opt in options" 
          :key="opt.value" 
          class="option" 
          :class="{ selected: opt.value === modelValue }"
          @click="select(opt.value)"
        >
          <span>{{ opt.label }}</span>
          <Check v-if="opt.value === modelValue" :size="16" class="check" />
        </div>
      </div>
    </transition>
    
    <!-- Backdrop to close -->
    <div v-if="isOpen" class="backdrop" @click="isOpen = false"></div>
  </div>
</template>

<style scoped>
.custom-select {
  position: relative;
  width: 100%;
}

.trigger {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: #fff;
  border-radius: 12px;
  font-size: 16px;
  color: #000;
  cursor: pointer;
  transition: background 0.2s;
}

.trigger:active {
  background: #f2f2f7;
}

.icon {
  color: #8e8e93;
  transition: transform 0.3s;
}

.open .icon {
  transform: rotate(180deg);
}

.dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  margin-top: 8px;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(20px);
  border-radius: 12px;
  padding: 8px;
  box-shadow: 0 10px 40px rgba(0,0,0,0.2);
  z-index: 100;
  transform-origin: top center;
  max-height: 200px;
  overflow-y: auto;
}

.option {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 12px;
  border-radius: 8px;
  font-size: 16px;
  color: #000;
  cursor: pointer;
}

.option.selected {
  background: #f2f2f7;
  font-weight: 500;
}

.option:active {
  background: #e5e5ea;
}

.check {
  color: #007aff;
}

.backdrop {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 90;
  cursor: default;
}

/* Animations */
.dropdown-enter-active,
.dropdown-leave-active {
  transition: all 0.2s cubic-bezier(0.2, 0.8, 0.2, 1);
}

.dropdown-enter-from,
.dropdown-leave-to {
  opacity: 0;
  transform: translateY(-10px) scale(0.95);
}
</style>
