<template>
  <div class="project-header">
    <div class="header-left">
      <img 
        :src="isDark ? './assets/icons/SVG/liveplay-icon-darkmode@web.svg' : './assets/icons/SVG/liveplay-icon-lightmode@web.svg'"
        alt="LivePlay"
        class="header-logo"
      />
      <h2 class="project-name">{{ currentProject?.name || t('project.noProject') }}</h2>
    </div>
  
    <div>
      <div class="digital-clock">{{ currentTime }}</div>
    </div>

    <div class="header-right">
      <div class="item-actions">
        <button class="action-btn" @click="handleImport" :disabled="!currentProject">
          <span class="material-symbols-rounded">audio_file</span>
          <span>{{ t('playlist.importAudio') }}</span>
        </button>
        <button class="action-btn" @click="handleAddGroup" :disabled="!currentProject">
          <span class="material-symbols-rounded">folder</span>
          <span>{{ t('playlist.addGroup') }}</span>
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { v4 as uuidv4 } from 'uuid';
import { DEFAULT_AUDIO_ITEM, DEFAULT_GROUP_ITEM } from '~/types/project';

const { currentProject, addItem, findItemByUuid, findItemByIndex } = useProject();
const { t } = useLocalization();
const { activeCues } = useAudioEngine();

const isDark = computed(() => currentProject.value?.theme.mode === 'dark');
const currentTime = ref('00:00:00');

const handleImport = async () => {
  if (!import.meta.client || !window.electronAPI || !currentProject.value) return;

  const filePaths = await window.electronAPI.selectAudioFiles();
  if (!filePaths || filePaths.length === 0) return;

  for (const filePath of filePaths) {
    await importAudioFile(filePath);
  }
};

const importAudioFile = async (sourcePath: string) => {
  if (!currentProject.value) return;

  try {
    const fileName = sourcePath.split(/[\\/]/).pop() || 'audio.mp3';
    const uuid = uuidv4();
    const destPath = `${currentProject.value.folderPath}/media/${fileName}`;
    
    // Copy file to media folder
    const copyResult = await window.electronAPI.copyFile(sourcePath, destPath);
    if (!copyResult.success) {
      console.error('Failed to copy file:', copyResult.error);
      return;
    }

    // Get audio duration
    const duration = await getAudioDuration(destPath);

    // Create audio item WITHOUT waveform (will be generated async via ffmpeg)
    const audioItem: AudioItem = {
      ...DEFAULT_AUDIO_ITEM,
      uuid,
      index: [currentProject.value.items.length],
      displayName: fileName.replace(/\.[^/.]+$/, ''), // Remove extension
      type: 'audio',
      mediaFileName: fileName,
      mediaPath: `media/${fileName}`, // Store relative path to project folder
      waveformPath: `${currentProject.value.folderPath}/waveforms/${uuid}.json`,
      waveform: undefined, // Will be generated asynchronously
      outPoint: duration,
      duration
    } as AudioItem;

    // Add item immediately (no blocking)
    addItem(audioItem);
    
    // Generate waveform asynchronously using ffmpeg
    generateWaveformAsync(audioItem);
  } catch (error) {
    console.error('Error importing audio:', error);
  }
};

const generateWaveformAsync = async (item: AudioItem) => {
  try {
    if (!currentProject.value) return;
    
    // Ensure waveforms directory exists
    const waveformsDir = `${currentProject.value.folderPath}/waveforms`;
    await window.electronAPI.ensureDirectory(waveformsDir);
    
    // Check if waveform file already exists and is valid
    const existingWaveform = await window.electronAPI.readFile(item.waveformPath);
    if (existingWaveform.success && existingWaveform.data) {
      try {
        const waveformData = JSON.parse(existingWaveform.data);
        
        // Validate waveform format (duration field is optional now)
        if (waveformData.peaks && waveformData.peaks.length > 0) {
          item.waveform = waveformData;
          
          // Update duration from waveform data if available (more accurate than Audio API)
          if (waveformData.duration && waveformData.duration > 0) {
            item.duration = waveformData.duration;
            item.outPoint = waveformData.duration;
          }
          
          triggerWaveformUpdate();
          console.log(`Existing waveform loaded for ${item.displayName}`);
          return;
        }
        console.warn('Invalid waveform format, regenerating...');
      } catch (e) {
        console.warn('Failed to parse existing waveform, regenerating...');
      }
    }
    
    // Check if generateWaveform is available
    if (!window.electronAPI.generateWaveform) {
      console.warn('generateWaveform not implemented yet - waveform will not be generated');
      console.info('Please implement the generateWaveform IPC handler in your Electron main process');
      return;
    }
    
    // Generate waveform using ffmpeg (non-blocking)
    const mediaPath = `${currentProject.value.folderPath}/media/${item.mediaFileName}`;
    const result = await window.electronAPI.generateWaveform(mediaPath, item.waveformPath);
    
    if (result.success) {
      console.log(`Started waveform generation for ${item.displayName}`);
      
      // Start polling for waveform file (check every 2 seconds)
      const pollInterval = setInterval(async () => {
        try {
          const waveformFile = await window.electronAPI.readFile(item.waveformPath);
          if (waveformFile.success && waveformFile.data) {
            const waveformData = JSON.parse(waveformFile.data);
            
            // Validate waveform format (duration field is optional)
            if (waveformData.peaks && waveformData.peaks.length > 0) {
              item.waveform = waveformData;
              
              // Update duration from waveform data if available (more accurate than Audio API)
              if (waveformData.duration && waveformData.duration > 0) {
                item.duration = waveformData.duration;
                item.outPoint = waveformData.duration;
              }
              
              // Force Vue reactivity update
              triggerWaveformUpdate();
              
              // Stop polling once loaded
              clearInterval(pollInterval);
              console.log(`Waveform loaded for ${item.displayName} (${waveformData.peaks.length} peaks, ${waveformData.duration?.toFixed(2)}s)`);
            }
          }
        } catch (error) {
          console.error('Error polling for waveform:', error);
        }
      }, 2000);
      
      // Stop polling after 30 seconds to prevent infinite polling
      setTimeout(() => {
        clearInterval(pollInterval);
      }, 30000);
    } else {
      console.error('Failed to generate waveform:', result.error);
    }
  } catch (error) {
    console.error('Error generating waveform:', error);
  }
};

const getAudioDuration = async (filePath: string): Promise<number> => {
  // Simplified - would use proper audio decoding
  return new Promise((resolve) => {
    if (import.meta.client) {
      const audio = new Audio(`file://${filePath}`);
      audio.addEventListener('loadedmetadata', () => {
        resolve(audio.duration);
      });
      audio.addEventListener('error', () => {
        resolve(60); // Default fallback
      });
    } else {
      resolve(60);
    }
  });
};

const handleAddGroup = () => {
  if (!currentProject.value) return;

  const groupItem: GroupItem = {
    ...DEFAULT_GROUP_ITEM,
    uuid: uuidv4(),
    index: [currentProject.value.items.length],
    displayName: 'New Group',
    type: 'group',
    children: [] // Create a new array for each group to avoid shared references
  } as GroupItem;

  addItem(groupItem);
};

const updateClock = () => {
  const now = new Date();
  const hours = now.getHours().toString().padStart(2, '0');
  const minutes = now.getMinutes().toString().padStart(2, '0');
  const seconds = now.getSeconds().toString().padStart(2, '0');
  currentTime.value = `${hours}:${minutes}:${seconds}`;
};

onMounted(() => {
  updateClock();
  const clockInterval = setInterval(updateClock, 1000);

  onUnmounted(() => {
    clearInterval(clockInterval);
  });
});
</script>

<style scoped lang="scss">
.project-header {
  position: relative; // For absolute positioning of warning
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--spacing-sm) var(--spacing-lg);
  background-color: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
  min-height: 60px;
}

.header-left {
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
}

.header-logo {
  width: 36px;
  height: 36px;
  object-fit: contain;
}

.project-name {
  font-size: 18px;
  font-weight: 600;
  color: var(--color-text-primary);
  margin: 0;
}

.header-right {
  display: flex;
  flex-direction: row;
  flex-wrap: nowrap;
}

.digital-clock {
  font-size: 24px;
  font-weight: 700;
  color: var(--color-accent);
  letter-spacing: 0.05em;
  padding: var(--spacing-xs) var(--spacing-md);
  border: 2px solid var(--color-accent);
  border-radius: var(--border-radius-md);
  background-color: var(--color-surface);
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  transition: all var(--transition-base);
}

.item-actions {
  display: flex;
  gap: var(--spacing-sm);
}

.action-btn {
  padding: var(--spacing-sm) var(--spacing-md);
  background-color: var(--color-background);
  border: 1px solid var(--color-border);
  border-radius: var(--border-radius-sm);
  font-size: 13px;
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  transition: all 0.2s;
  
  &:hover:not(:disabled) {
    background-color: var(--color-surface-hover);
    border-color: var(--color-accent);
  }
  
  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
}
</style>
