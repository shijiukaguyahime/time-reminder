<script setup lang="ts">
import { getCurrentWindow } from '@tauri-apps/api/window';
import { X, Minus, Settings } from 'lucide-vue-next';

const emit = defineEmits<{
  (e: 'open-settings'): void;
}>();

const appWindow = getCurrentWindow();

// Read preference from localStorage (synced via props or simple read)
// For now, simple read. In App.vue we might control this better, but titlebar is independent.
// Actually, App.vue should pass this behavior or we read it here.
// Let's assume standard behavior for now, but I'll add the setting logic later.

function minimize() {
  appWindow.minimize();
}

async function close() {
  const closeAction = localStorage.getItem('closeAction') || 'minimize';
  if (closeAction === 'exit') {
    appWindow.close();
  } else {
    // Minimize to tray means hiding the window
    appWindow.hide();
  }
}

function openSettings() {
  emit('open-settings');
}
</script>

<template>
  <div data-tauri-drag-region class="titlebar">
    <div class="left-controls">
      <button class="control-btn settings" @click="openSettings" title="设置">
        <Settings :size="16" />
      </button>
    </div>
    <div data-tauri-drag-region class="title">定时提醒</div>
    <div class="controls">
      <button class="control-btn" @click="minimize">
        <Minus :size="16" />
      </button>
      <button class="control-btn close" @click="close">
        <X :size="16" />
      </button>
    </div>
  </div>
</template>

<style scoped>
.titlebar {
  height: 40px; /* Slightly taller for better touch */
  background: #fff;
  display: flex;
  justify-content: space-between;
  align-items: center;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  z-index: 1000;
  padding: 0 10px;
  user-select: none;
  width: 100%;
  box-sizing: border-box;
  -webkit-app-region: drag; /* 必须设置 */
  pointer-events: auto; /* 确保不是 none */
}

/* Ensure drag region covers the empty space */
.titlebar::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: -1;
}

.title {
  font-size: 16px;
  font-weight: 600;
  color: #000;
  pointer-events: none;
  flex: 1;
  text-align: center;
}

.controls, .left-controls {
  display: flex;
  gap: 8px;
  z-index: 2; /* Above drag layer */
  width: 60px; /* Balance layout */
}

.controls {
  justify-content: flex-end;
}

.control-btn {
  background: transparent;
  border: none;
  cursor: pointer;
  padding: 6px;
  border-radius: 50%; /* Round buttons */
  display: flex;
  align-items: center;
  justify-content: center;
  color: #666;
  transition: background 0.2s, color 0.2s;
}

.control-btn:hover {
  background: rgba(0, 0, 0, 0.05);
  color: #000;
}

.control-btn.close:hover {
  background: #ff4d4f;
  color: white;
}
</style>
