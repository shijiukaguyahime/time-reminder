<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';
import { Plus, Check, X } from 'lucide-vue-next';
import dayjs from 'dayjs';
import { isPermissionGranted, requestPermission, sendNotification } from '@tauri-apps/plugin-notification';
import { getCurrentWindow } from '@tauri-apps/api/window';
import CustomTitleBar from './components/CustomTitleBar.vue';
import TimePicker from './components/TimePicker.vue';
import AlarmItem from './components/AlarmItem.vue';
import CustomSelect from './components/CustomSelect.vue';

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
    customDays: number[]; // 0 (Sun) - 6 (Sat)
    shift?: {
      startDate: string; // YYYY-MM-DD
      workDays: number;
      restDays: number;
    };
  };
  lastTriggered?: string; // YYYY-MM-DD
}

interface AppSettings {
  closeAction: 'minimize' | 'exit';
  notificationStyle: 'system' | 'theme';
}

// --- State ---
const alarms = ref<Alarm[]>([]);
const isEditing = ref(false);
const editingId = ref<string | null>(null);
const isSettingsOpen = ref(false);
const showThemeNotification = ref(false);
const notificationData = ref({ title: '', body: '' });

// Selection Mode State
const isSelectionMode = ref(false);
const selectedIds = ref<Set<string>>(new Set());

// Add/Edit Mode State
const timeMode = ref<'absolute' | 'relative'>('absolute');
const relativeTime = ref({ hour: 1, minute: 1 });

// Settings State
const settings = ref<AppSettings>({
  closeAction: 'minimize',
  notificationStyle: 'system'
});

// Edit Form State
const form = ref({
  hour: 8,
  minute: 0,
  label: '定时提醒',
  repeatType: 'once' as RepeatType,
  customDays: [] as number[],
  shiftStart: dayjs().format('YYYY-MM-DD'),
  shiftWork: 1,
  shiftRest: 1,
});

// --- Constants ---
const REPEAT_OPTIONS: { label: string; value: RepeatType }[] = [
  { label: '仅一次', value: 'once' },
  { label: '每天', value: 'daily' },
  { label: '工作日 (周一至周五)', value: 'workdays' },
  { label: '工作日 (周一至周六)', value: 'mon_sat' },
  { label: '自定义周', value: 'custom' },
  { label: '轮班/倒班', value: 'shift' },
];

const WEEKDAYS = [
  { label: '周日', value: 0 },
  { label: '周一', value: 1 },
  { label: '周二', value: 2 },
  { label: '周三', value: 3 },
  { label: '周四', value: 4 },
  { label: '周五', value: 5 },
  { label: '周六', value: 6 },
];

// --- Lifecycle ---
onMounted(() => {
  loadAlarms();
  loadSettings();
  requestNotificationPermission();
  startScheduler();
});

// Persist alarms
watch(alarms, () => {
  saveAlarms();
}, { deep: true });

// Persist settings
watch(settings, () => {
  saveSettings();
}, { deep: true });

// Watch relative time changes to update form
watch(() => [relativeTime.value, timeMode.value], () => {
  if (timeMode.value === 'relative' && isEditing.value) {
    const now = dayjs();
    const target = now.add(relativeTime.value.hour, 'hour').add(relativeTime.value.minute, 'minute');
    form.value.hour = target.hour();
    form.value.minute = target.minute();
  }
}, { deep: true });

// --- Logic ---

function loadAlarms() {
  const stored = localStorage.getItem('alarms');
  if (stored) {
    alarms.value = JSON.parse(stored);
  } else {
    // Default demo alarm
    alarms.value = [{
      id: crypto.randomUUID(),
      hour: 8,
      minute: 30,
      label: '起床喝水',
      enabled: true,
      repeat: { type: 'daily', customDays: [] }
    }];
  }
}

function saveAlarms() {
  localStorage.setItem('alarms', JSON.stringify(alarms.value));
}

function loadSettings() {
  const stored = localStorage.getItem('settings');
  if (stored) {
    settings.value = JSON.parse(stored);
  }
  // Sync closeAction to localStorage for CustomTitleBar to read directly if needed,
  // or we can manage it entirely here if we move close logic to App (but TitleBar is component).
  // For simplicity, we write to a specific key the TitleBar reads.
  localStorage.setItem('closeAction', settings.value.closeAction);
}

function saveSettings() {
  localStorage.setItem('settings', JSON.stringify(settings.value));
  localStorage.setItem('closeAction', settings.value.closeAction);
}

async function requestNotificationPermission() {
  try {
    let granted = await isPermissionGranted();
    console.log('Initial permission check:', granted);
    
    if (!granted) {
      const permission = await requestPermission();
      granted = permission === 'granted';
      console.log('Permission request result:', permission);
    }
    
    if (granted) {
      console.log('Notification permission granted.');
    } else {
      console.error('Notification permission denied.');
    }
  } catch (error) {
    console.error('Error requesting notification permission:', error);
  }
}

function startScheduler() {
  let lastRunMinute = '';

  setInterval(() => {
    const now = dayjs();
    const currentMinuteStr = now.format('YYYY-MM-DD HH:mm');
    
    // Run only once per minute
    if (currentMinuteStr === lastRunMinute) return;
    lastRunMinute = currentMinuteStr;

    const currentHour = now.hour();
    const currentMinute = now.minute();
    const todayStr = now.format('YYYY-MM-DD');
    const currentDay = now.day(); // 0-6

    alarms.value.forEach(alarm => {
      if (!alarm.enabled) return;

      // Check if time matches
      if (alarm.hour === currentHour && alarm.minute === currentMinute) {
        // Check repeat logic
        if (shouldTrigger(alarm, now, todayStr, currentDay)) {
          triggerAlarm(alarm);
          
          // Handle 'once' logic: disable after trigger
          if (alarm.repeat.type === 'once') {
            alarm.enabled = false;
          }
          
          alarm.lastTriggered = todayStr;
        }
      }
    });
  }, 1000);
}

function shouldTrigger(alarm: Alarm, now: dayjs.Dayjs, todayStr: string, currentDay: number): boolean {
  // Prevent double trigger in same minute (though interval checks seconds, safety net)
  if (alarm.lastTriggered === todayStr && alarm.repeat.type === 'once') return false;

  const { type, customDays, shift } = alarm.repeat;

  if (type === 'once') {
    return true;
  }
  
  if (type === 'daily') return true;
  
  if (type === 'workdays') return currentDay >= 1 && currentDay <= 5;
  
  if (type === 'mon_sat') return currentDay >= 1 && currentDay <= 6;
  
  if (type === 'custom') return customDays.includes(currentDay);
  
  if (type === 'shift' && shift) {
    const start = dayjs(shift.startDate);
    const diffDays = now.diff(start, 'day');
    if (diffDays < 0) return false; // Before start date
    
    const cycleLength = shift.workDays + shift.restDays;
    const positionInCycle = diffDays % cycleLength;
    
    return positionInCycle < shift.workDays; // Trigger on work days
  }

  return false;
}

async function triggerAlarm(alarm: Alarm) {
  const title = '定时提醒';
  const body = `${alarm.label || '时间到了'} - ${pad(alarm.hour)}:${pad(alarm.minute)}`;
  
  // 1. Check settings first
  if (settings.value.notificationStyle === 'theme') {
     notificationData.value = { title, body };
     showThemeNotification.value = true;
  } else {
     // 2. Send system notification (Optimistic approach)
     try {
       await sendNotification({
         title: title,
         body: body,
       });
     } catch (e) {
       console.warn('System notification failed, retrying with permission check...', e);
       
       // Fallback: check permission
       try {
         let granted = await isPermissionGranted();
         if (!granted) {
           const permission = await requestPermission();
           granted = permission === 'granted';
         }
   
         if (granted) {
            await sendNotification({
              title: title,
              body: body,
            });
         } else {
            // Permission denied, fallback to theme
            notificationData.value = { title, body };
            showThemeNotification.value = true;
         }
       } catch (retryError) {
         console.error('Retry failed:', retryError);
         // Final fallback
         notificationData.value = { 
           title, 
           body: `${body} (系统通知失败)` 
         };
         showThemeNotification.value = true;
       }
     }
  }

  // 3. Handle window visibility
  try {
    const appWindow = getCurrentWindow();
    await appWindow.unminimize();
    await appWindow.show();
    await appWindow.setFocus();
  } catch (e) {
    console.error('Failed to focus window:', e);
  }
}

function pad(n: number) {
  return n.toString().padStart(2, '0');
}

function closeThemeNotification() {
  showThemeNotification.value = false;
}

// --- Selection Actions ---

function handleLongPress(id: string) {
  if (isSelectionMode.value) return;
  isSelectionMode.value = true;
  selectedIds.value.clear();
  selectedIds.value.add(id);
}

function handleSelect(id: string) {
  if (selectedIds.value.has(id)) {
    selectedIds.value.delete(id);
    if (selectedIds.value.size === 0) {
      // Optional: Exit mode if nothing selected?
      // For now keep mode open until user cancels or deletes
    }
  } else {
    selectedIds.value.add(id);
  }
}

function deleteSelected() {
  if (selectedIds.value.size === 0) {
    isSelectionMode.value = false;
    return;
  }
  
  alarms.value = alarms.value.filter(a => !selectedIds.value.has(a.id));
  isSelectionMode.value = false;
  selectedIds.value.clear();
}

function cancelSelection() {
  isSelectionMode.value = false;
  selectedIds.value.clear();
}

// --- UI Actions ---

function openAdd() {
  editingId.value = null;
  form.value = {
    hour: dayjs().hour(),
    minute: dayjs().minute(),
    label: '',
    repeatType: 'once',
    customDays: [],
    shiftStart: dayjs().format('YYYY-MM-DD'),
    shiftWork: 1,
    shiftRest: 1,
  };
  isEditing.value = true;
}

function openEdit(id: string) {
  const alarm = alarms.value.find(a => a.id === id);
  if (!alarm) return;
  
  editingId.value = id;
  form.value = {
    hour: alarm.hour,
    minute: alarm.minute,
    label: alarm.label,
    repeatType: alarm.repeat.type,
    customDays: [...alarm.repeat.customDays],
    shiftStart: alarm.repeat.shift?.startDate || dayjs().format('YYYY-MM-DD'),
    shiftWork: alarm.repeat.shift?.workDays || 1,
    shiftRest: alarm.repeat.shift?.restDays || 1,
  };
  isEditing.value = true;
}

function saveForm() {
  const finalLabel = form.value.label.trim() ? form.value.label : '定时提醒';
  const alarmData: Alarm = {
    id: editingId.value || crypto.randomUUID(),
    hour: form.value.hour,
    minute: form.value.minute,
    label: finalLabel,
    enabled: true, // Re-enable on edit
    repeat: {
      type: form.value.repeatType,
      customDays: form.value.customDays,
      shift: form.value.repeatType === 'shift' ? {
        startDate: form.value.shiftStart,
        workDays: Number(form.value.shiftWork),
        restDays: Number(form.value.shiftRest),
      } : undefined
    }
  };

  if (editingId.value) {
    const index = alarms.value.findIndex(a => a.id === editingId.value);
    if (index !== -1) alarms.value[index] = alarmData;
  } else {
    alarms.value.push(alarmData);
  }
  
  isEditing.value = false;
}

function deleteAlarm() {
  if (editingId.value) {
    alarms.value = alarms.value.filter(a => a.id !== editingId.value);
  }
  isEditing.value = false;
}

function toggleAlarm(id: string, enabled: boolean) {
  const alarm = alarms.value.find(a => a.id === id);
  if (alarm) alarm.enabled = enabled;
}

function toggleCustomDay(day: number) {
  const index = form.value.customDays.indexOf(day);
  if (index === -1) {
    form.value.customDays.push(day);
  } else {
    form.value.customDays.splice(index, 1);
  }
}

async function testNotification() {
  console.log('Testing notification...');
  try {
    await sendNotification({
      title: '测试提醒',
      body: '这是一条测试通知',
    });
    console.log('Test notification sent.');
  } catch (e) {
    console.error('Failed to send test notification:', e);
  }
}

// --- Settings ---
function openSettings() {
  isSettingsOpen.value = true;
}

function closeSettings() {
  isSettingsOpen.value = false;
}
</script>

<template>
  <div class="app-container">
    <CustomTitleBar @open-settings="openSettings" />
    
    <!-- Main List View -->
    <div class="content" v-show="!isEditing && !isSettingsOpen" @click="isSelectionMode ? cancelSelection() : null">
      <div class="header">
        <h1>定时提醒</h1>
      </div>
      
      <div class="alarm-list">
        <AlarmItem 
          v-for="alarm in alarms" 
          :key="alarm.id" 
          :alarm="alarm" 
          :is-selection-mode="isSelectionMode"
          :is-selected="selectedIds.has(alarm.id)"
          @toggle="toggleAlarm" 
          @edit="openEdit"
          @longpress="handleLongPress"
          @select="handleSelect"
        />
        <div v-if="alarms.length === 0" class="empty-state">
          没有提醒，点击下方 + 号添加
        </div>
      </div>
      
      <div class="fab-container">
        <button 
          class="fab" 
          :class="{ delete: isSelectionMode }"
          @click="isSelectionMode ? deleteSelected() : openAdd()"
        >
          <transition name="rotate" mode="out-in">
             <component :is="isSelectionMode ? X : Plus" :size="24" color="white" />
          </transition>
        </button>
      </div>
    </div>
    
    <!-- Edit/Add View (Modal-like) -->
    <div class="edit-view" v-if="isEditing">
      <div class="edit-header">
        <button class="nav-btn" @click="isEditing = false">取消</button>
        <span class="edit-title">{{ editingId ? '编辑提醒' : '添加提醒' }}</span>
        <button class="nav-btn save" @click="saveForm">保存</button>
      </div>
      
      <div class="edit-body">
        <TimePicker 
          v-model:hour="form.hour" 
          v-model:minute="form.minute" 
        />
        
        <div class="form-group">
          <label>标签</label>
          <input v-model="form.label" placeholder="定时提醒" />
        </div>
        
        <div class="form-group no-padding">
          <CustomSelect 
            v-model="form.repeatType" 
            :options="REPEAT_OPTIONS" 
          />
        </div>
        
        <!-- Custom Days Selector -->
        <div class="form-group days-selector" v-if="form.repeatType === 'custom'">
          <div 
            v-for="day in WEEKDAYS" 
            :key="day.value" 
            class="day-btn" 
            :class="{ active: form.customDays.includes(day.value) }"
            @click="toggleCustomDay(day.value)"
          >
            {{ day.label }}
          </div>
        </div>
        
        <!-- Shift Settings -->
        <div class="shift-settings" v-if="form.repeatType === 'shift'">
          <div class="form-group">
            <label>开始日期</label>
            <input type="date" v-model="form.shiftStart" />
          </div>
          <div class="form-row">
            <div class="form-group half">
              <label>上班天数</label>
              <input type="number" v-model="form.shiftWork" min="1" />
            </div>
            <div class="form-group half">
              <label>休息天数</label>
              <input type="number" v-model="form.shiftRest" min="1" />
            </div>
          </div>
        </div>
        
        <button v-if="editingId" class="delete-btn" @click="deleteAlarm">
          删除提醒
        </button>
      </div>
    </div>

    <!-- Settings View -->
    <div class="edit-view" v-if="isSettingsOpen">
      <div class="edit-header">
        <button class="nav-btn" @click="closeSettings">返回</button>
        <span class="edit-title">设置</span>
        <div style="width: 40px;"></div> <!-- Spacer -->
      </div>
      <div class="edit-body">
        <div class="section-title">通用</div>
        <div class="form-group no-padding">
          <div class="setting-item">
            <span>关闭时</span>
            <div class="setting-control">
              <button 
                class="segment-btn" 
                :class="{ active: settings.closeAction === 'minimize' }"
                @click="settings.closeAction = 'minimize'"
              >
                最小化
              </button>
              <button 
                class="segment-btn" 
                :class="{ active: settings.closeAction === 'exit' }"
                @click="settings.closeAction = 'exit'"
              >
                退出
              </button>
            </div>
          </div>
        </div>

        <div class="section-title">通知</div>
        <div class="form-group no-padding">
          <div class="setting-item">
            <span>弹窗样式</span>
            <div class="setting-control">
              <button 
                class="segment-btn" 
                :class="{ active: settings.notificationStyle === 'system' }"
                @click="settings.notificationStyle = 'system'"
              >
                系统
              </button>
              <button 
                class="segment-btn" 
                :class="{ active: settings.notificationStyle === 'theme' }"
                @click="settings.notificationStyle = 'theme'"
              >
                主题
              </button>
            </div>
          </div>
        </div>
        
        <button class="delete-btn" style="color: #007aff; margin-top: 10px;" @click="testNotification">
          测试系统通知
        </button>
      </div>
    </div>

    <!-- Theme Notification Modal -->
    <transition name="fade">
      <div v-if="showThemeNotification" class="notification-overlay">
        <div class="notification-card">
          <div class="notif-icon">
            <Check :size="32" color="white" />
          </div>
          <h3>{{ notificationData.title }}</h3>
          <p>{{ notificationData.body }}</p>
          <button @click="closeThemeNotification">知道了</button>
        </div>
      </div>
    </transition>

  </div>
</template>

<style>
html, body {
  margin: 0;
  padding: 0;
  background: transparent !important; /* Ensure transparency */
  overflow: hidden;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  width: 100vw;
  height: 100vh;
}

#app {
  width: 100%;
  height: 100%;
}
</style>

<style scoped>
/* Main Layout */
.app-container {
  width: 100%;
  height: 100%;
  background: #f2f2f7; /* iOS background gray */
  display: flex;
  flex-direction: column;
  overflow: hidden;
  /* border-radius: 20px; */ /* Removed to fix potential white edge artifacts */
  /* box-shadow: 0 0 20px rgba(0,0,0,0.2); */
  position: relative;
}

.content {
  flex: 1;
  padding: 50px 20px 100px; /* Top padding for titlebar */
  overflow-y: auto;
}

.header h1 {
  font-size: 34px;
  font-weight: 700;
  color: #000;
  margin-bottom: 20px;
}

.empty-state {
  text-align: center;
  color: #8e8e93;
  margin-top: 100px;
}

/* FAB */
.fab-container {
  position: fixed;
  bottom: 30px;
  left: 0;
  right: 0;
  display: flex;
  justify-content: center;
  pointer-events: none;
}

.fab {
  pointer-events: auto;
  width: 60px;
  height: 60px;
  border-radius: 30px;
  background: #007aff; /* iOS Blue */
  border: none;
  box-shadow: 0 4px 12px rgba(0, 122, 255, 0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: transform 0.2s, background-color 0.3s ease;
}

.fab:active {
  transform: scale(0.9);
}

.fab.delete {
  background: #ff3b30;
}

.rotate-enter-active,
.rotate-leave-active {
  transition: transform 0.3s ease;
}

.rotate-enter-from {
  transform: rotate(-90deg);
  opacity: 0;
}

.rotate-leave-to {
  transform: rotate(90deg);
  opacity: 0;
}

/* Edit View */
.edit-view {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #f2f2f7;
  z-index: 100;
  display: flex;
  flex-direction: column;
  animation: slideUp 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  border-radius: 20px;
  overflow: hidden;
}

@keyframes slideUp {
  from { transform: translateY(100%); }
  to { transform: translateY(0); }
}

.edit-header {
  padding: 40px 16px 16px; /* Top padding for safe area/titlebar */
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #f2f2f7;
}

.edit-title {
  font-weight: 600;
  font-size: 17px;
}

.nav-btn {
  background: none;
  border: none;
  font-size: 17px;
  color: #007aff;
  cursor: pointer;
}

.nav-btn.save {
  font-weight: 600;
}

.edit-body {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
}

/* Form Styles */
/* Relative Time Picker Styles */
.relative-time-picker {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px 0;
  background: white;
  border-radius: 12px;
  margin-bottom: 20px;
}

.relative-inputs {
  display: flex;
  gap: 20px;
  margin-bottom: 16px;
}

.input-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

.input-group input {
  width: 60px;
  height: 40px;
  text-align: center;
  font-size: 18px;
  border: 1px solid #e5e5ea;
  border-radius: 8px;
  background: #f2f2f7;
}

.input-group span {
  font-size: 16px;
  color: #1c1c1e;
}

.preview-time {
  font-size: 14px;
  color: #8e8e93;
}

.form-group {
  background: white;
  padding: 12px 16px;
  border-radius: 10px;
  margin-bottom: 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.form-group.no-padding {
  padding: 0;
  background: transparent;
}

.form-group label {
  font-size: 17px;
  color: #000;
}

.form-group input {
  border: none;
  text-align: right;
  font-size: 17px;
  color: #8e8e93;
  background: transparent;
  outline: none;
  min-width: 100px;
}

.days-selector {
  flex-wrap: wrap;
  justify-content: flex-start;
  gap: 8px;
}

.day-btn {
  padding: 8px 12px;
  background: #f2f2f7;
  border-radius: 20px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s;
}

.day-btn.active {
  background: #007aff;
  color: white;
}

.form-row {
  display: flex;
  gap: 12px;
}

.form-group.half {
  flex: 1;
}

.delete-btn {
  width: 100%;
  padding: 16px;
  background: white;
  color: #ff3b30;
  border: none;
  border-radius: 10px;
  font-size: 17px;
  font-weight: 600;
  margin-top: 20px;
  cursor: pointer;
}

.delete-btn:active {
  background: #f2f2f7;
}

/* Settings Styles */
.section-title {
  font-size: 14px;
  color: #8e8e93;
  margin: 20px 0 8px 16px;
  text-transform: uppercase;
}

.setting-item {
  background: white;
  border-radius: 10px;
  padding: 12px 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.setting-control {
  background: #f2f2f7;
  padding: 2px;
  border-radius: 8px;
  display: flex;
}

.segment-btn {
  padding: 6px 12px;
  border: none;
  background: transparent;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  color: #000;
}

.segment-btn.active {
  background: white;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  font-weight: 600;
}

/* Notification Overlay */
.notification-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.5);
  z-index: 2000;
  display: flex;
  align-items: center;
  justify-content: center;
  backdrop-filter: blur(5px);
  border-radius: 20px;
  overflow: hidden;
}

.notification-card {
  background: white;
  width: 80%;
  max-width: 300px;
  border-radius: 20px;
  padding: 30px;
  text-align: center;
  box-shadow: 0 20px 50px rgba(0,0,0,0.3);
  animation: popIn 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

@keyframes popIn {
  from { transform: scale(0.8); opacity: 0; }
  to { transform: scale(1); opacity: 1; }
}

.notif-icon {
  width: 60px;
  height: 60px;
  background: #34c759;
  border-radius: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 20px;
}

.notification-card h3 {
  margin: 0 0 10px;
  font-size: 20px;
}

.notification-card p {
  color: #666;
  margin: 0 0 20px;
}

.notification-card button {
  background: #007aff;
  color: white;
  border: none;
  padding: 10px 30px;
  border-radius: 20px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
