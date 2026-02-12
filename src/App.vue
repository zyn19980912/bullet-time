<template>
  <div class="header">
    <h1>子弹时间</h1>
    <p>本地语音笔记 · 随时记录</p>
  </div>

  <div class="record-area">
    <button 
      class="mic-button" 
      :class="{ recording: isRecording }"
      @mousedown="startRecording"
      @mouseup="stopRecording"
      @mouseleave="cancelRecording"
      @touchstart.prevent="startRecording"
      @touchend.prevent="stopRecording"
      @touchcancel.prevent="cancelRecording"
    >
      <svg class="mic-icon" viewBox="0 0 24 24">
        <path d="M12 14c1.66 0 3-1.34 3-3V5c0-1.66-1.34-3-3-3S9 3.34 9 5v6c0 1.66 1.34 3 3 3z"/>
        <path d="M17 11c0 2.76-2.24 5-5 5s-5-2.24-5-5H5c0 3.53 2.61 6.43 6 6.92V21h2v-3.08c3.39-.49 6-3.39 6-6.92h-2z"/>
      </svg>
    </button>
    <div class="status-text">{{ statusText }}</div>
  </div>

  <div v-if="showTranscribePopup" class="overlay" @click="closeTranscribePopup"></div>
  <div v-if="showTranscribePopup" class="transcribe-popup">
    <div class="transcribe-title">录音已保存</div>
    <div class="transcribe-buttons">
      <button class="btn btn-secondary" @click="closeTranscribePopup">稍后处理</button>
      <button class="btn btn-primary" @click="transcribe">转为文字</button>
    </div>
  </div>

  <div class="notes-list">
    <div class="notes-title">我的笔记 ({{ notes.length }})</div>
    
    <div v-if="notes.length === 0" class="empty-state">
      <svg viewBox="0 0 24 24" fill="currentColor">
        <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 15l-5-5 1.41-1.41L10 14.17l7.59-7.59L19 8l-9 9z"/>
      </svg>
      <p>还没有笔记，点击上方麦克风开始录音</p>
    </div>

    <div v-for="note in notes" :key="note.id" class="note-item">
      <div class="note-header">
        <span class="note-time">{{ formatTime(note.timestamp) }}</span>
        <div class="note-actions">
          <button 
            class="note-action-btn" 
            @click="copyText(note)"
            v-if="note.transcribedText"
          >
            Copy
          </button>
          <button 
            class="note-action-btn" 
            @click="transcribeNote(note)"
            v-if="!note.transcribedText"
          >
            Transcribe
          </button>
          <button class="note-action-btn delete" @click="deleteNote(note.id)">Delete</button>
        </div>
      </div>
      
      <div v-if="note.transcribedText !== undefined">
        <textarea
          v-if="note.isEditing"
          v-model="note.transcribedText"
          class="note-content editing"
          @blur="saveEdit(note)"
          @keyup.enter="saveEdit(note)"
          rows="4"
        />
        <div v-else class="note-content" @click="startEdit(note)">
          {{ note.transcribedText || 'Click to transcribe' }}
        </div>
      </div>
      
      <audio :src="note.audioUrl" controls class="audio-player"></audio>
    </div>
  </div>

  <div v-if="toast" class="toast">{{ toast }}</div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const isRecording = ref(false)
const showTranscribePopup = ref(false)
const notes = ref([])
const toast = ref('')
const currentRecording = ref(null)
const lastRecordingBlob = ref(null)

const mediaRecorder = ref(null)
const audioChunks = ref([])
const recordStartTime = ref(0)
const recordTimer = ref(null)

let db = null
const statusText = ref('Click to start recording')

const initDB = () => {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('BulletTimeDB', 1)
    
    request.onerror = () => reject(request.error)
    request.onsuccess = () => {
      db = request.result
      resolve(db)
    }
    
    request.onupgradeneeded = (event) => {
      db = event.target.result
      if (!db.objectStoreNames.contains('notes')) {
        const store = db.createObjectStore('notes', { keyPath: 'id' })
        store.createIndex('timestamp', 'timestamp', { unique: false })
      }
    }
  })
}

const loadNotes = () => {
  const transaction = db.transaction(['notes'], 'readonly')
  const store = transaction.objectStore('notes')
  const index = store.index('timestamp')
  const request = index.openCursor(null, 'prev')
  
  const loadedNotes = []
  request.onsuccess = (event) => {
    const cursor = event.target.result
    if (cursor) {
      loadedNotes.push({
        ...cursor.value,
        audioUrl: URL.createObjectURL(new Blob([cursor.value.audioData], { type: 'audio/webm' }))
      })
      cursor.continue()
    } else {
      notes.value = loadedNotes
    }
  }
}

const saveNote = async (blob) => {
  const audioData = await blob.arrayBuffer()
  const note = {
    id: Date.now(),
    timestamp: Date.now(),
    audioData: audioData,
    transcribedText: undefined,
    isEditing: false
  }
  
  const transaction = db.transaction(['notes'], 'readwrite')
  const store = transaction.objectStore('notes')
  store.add(note)
  
  currentRecording.value = note
  return note
}

const startRecording = async () => {
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
    
    mediaRecorder.value = new MediaRecorder(stream)
    audioChunks.value = []
    
    mediaRecorder.value.ondataavailable = (event) => {
      audioChunks.value.push(event.data)
    }
    
    mediaRecorder.value.onstop = async () => {
      lastRecordingBlob.value = new Blob(audioChunks.value, { type: 'audio/webm' })
      await saveNote(lastRecordingBlob.value)
      showTranscribePopup.value = true
    }
    
    mediaRecorder.value.start()
    isRecording.value = true
    recordStartTime.value = Date.now()
    statusText.value = 'Recording... Release to save'
    
    recordTimer.value = setInterval(() => {
      const duration = Math.floor((Date.now() - recordStartTime.value) / 1000)
      const minutes = Math.floor(duration / 60).toString().padStart(2, '0')
      const seconds = (duration % 60).toString().padStart(2, '0')
      statusText.value = `Recording... ${minutes}:${seconds}`
    }, 1000)
    
  } catch (error) {
    showToast('Cannot access microphone')
    console.error('Recording error:', error)
  }
}

const stopRecording = () => {
  if (isRecording.value && mediaRecorder.value) {
    mediaRecorder.value.stop()
    mediaRecorder.value.stream.getTracks().forEach(track => track.stop())
    isRecording.value = false
    clearInterval(recordTimer.value)
    statusText.value = 'Click to start recording'
  }
}

const cancelRecording = () => {
  if (isRecording.value && mediaRecorder.value) {
    mediaRecorder.value.stream.getTracks().forEach(track => track.stop())
    isRecording.value = false
    clearInterval(recordTimer.value)
    statusText.value = 'Click to start recording'
  }
}

const closeTranscribePopup = async () => {
  showTranscribePopup.value = false
  await loadNotes()
}

const transcribe = async () => {
  if (!currentRecording.value) return
  
  showTranscribePopup.value = false
  showToast('Recognizing...')
  
  const note = currentRecording.value
  
  if (!('webkitSpeechRecognition' in window) && !('SpeechRecognition' in window)) {
    showToast('Browser does not support speech recognition')
    return
  }
  
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
  const recognition = new SpeechRecognition()
  
  recognition.lang = 'zh-CN'
  recognition.continuous = true
  recognition.interimResults = false
  
  const audioBlob = lastRecordingBlob.value
  const audioUrl = URL.createObjectURL(audioBlob)
  const audio = new Audio(audioUrl)
  
  recognition.start()
  audio.play()
  
  recognition.onresult = (event) => {
    let finalTranscript = ''
    for (let i = event.resultIndex; i < event.results.length; i++) {
      if (event.results[i].isFinal) {
        finalTranscript += event.results[i][0].transcript
      }
    }
    
    if (finalTranscript) {
      updateNoteTranscription(note.id, finalTranscript)
      showToast('Recognition successful')
    } else {
      showToast('No content detected, please try again')
    }
  }
  
  recognition.onerror = (event) => {
    console.error('Recognition error:', event.error)
    showToast('Recognition failed: ' + event.error)
  }
  
  recognition.onend = () => {
    audio.pause()
    audio.currentTime = 0
  }
}

const updateNoteTranscription = (noteId, text) => {
  const transaction = db.transaction(['notes'], 'readwrite')
  const store = transaction.objectStore('notes')
  const request = store.get(noteId)
  
  request.onsuccess = () => {
    const note = request.result
    note.transcribedText = text
    store.put(note)
    loadNotes()
  }
}

const transcribeNote = async (note) => {
  if (!('webkitSpeechRecognition' in window) && !('SpeechRecognition' in window)) {
    showToast('Browser does not support speech recognition')
    return
  }
  
  showToast('Recognizing...')
  
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
  const recognition = new SpeechRecognition()
  
  recognition.lang = 'zh-CN'
  recognition.continuous = true
  recognition.interimResults = false
  
  const audio = new Audio(note.audioUrl)
  
  recognition.start()
  audio.play()
  
  recognition.onresult = (event) => {
    let finalTranscript = ''
    for (let i = event.resultIndex; i < event.results.length; i++) {
      if (event.results[i].isFinal) {
        finalTranscript += event.results[i][0].transcript
      }
    }
    
    if (finalTranscript) {
      updateNoteTranscription(note.id, finalTranscript)
      showToast('Recognition successful')
    } else {
      showToast('No content detected, please try again')
    }
  }
  
  recognition.onerror = (event) => {
    console.error('Recognition error:', event.error)
    showToast('Recognition failed: ' + event.error)
  }
  
  recognition.onend = () => {
    audio.pause()
    audio.currentTime = 0
  }
}

const copyText = async (note) => {
  if (!note.transcribedText) return
  
  try {
    await navigator.clipboard.writeText(note.transcribedText)
    showToast('Copied to clipboard')
  } catch (error) {
    showToast('Copy failed')
  }
}

const startEdit = (note) => {
  if (note.transcribedText) {
    note.isEditing = true
  }
}

const saveEdit = async (note) => {
  note.isEditing = false
  
  const transaction = db.transaction(['notes'], 'readwrite')
  const store = transaction.objectStore('notes')
  const request = store.get(note.id)
  
  request.onsuccess = () => {
    const data = request.result
    data.transcribedText = note.transcribedText
    store.put(data)
  }
}

const deleteNote = (noteId) => {
  if (!confirm('Delete this note?')) return
  
  const transaction = db.transaction(['notes'], 'readwrite')
  const store = transaction.objectStore('notes')
  store.delete(noteId)
  
  notes.value = notes.value.filter(note => note.id !== noteId)
  showToast('Deleted')
}

const formatTime = (timestamp) => {
  const date = new Date(timestamp)
  const now = new Date()
  const diff = now - date
  
  if (diff < 60000) return 'Just now'
  if (diff < 3600000) return Math.floor(diff / 60000) + ' mins ago'
  if (diff < 86400000) return Math.floor(diff / 3600000) + ' hours ago'
  
  return date.toLocaleDateString('zh-CN', {
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  })
}

const showToast = (message) => {
  toast.value = message
  setTimeout(() => {
    toast.value = ''
  }, 2000)
}

onMounted(async () => {
  await initDB()
  loadNotes()
})
</script>
