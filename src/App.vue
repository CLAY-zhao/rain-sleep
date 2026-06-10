<!-- src/App.vue -->
<template>
  <div class="app-container">
    <!-- 动态背景 -->
    <div class="bg-gradient"></div>
    <div class="bg-rain"></div>

    <!-- 主内容 -->
    <div class="content">
      <!-- 头部设计 -->
      <header class="header">
        <div class="header-glow"></div>
        <div class="moon-phase">
          <div class="moon-icon">
            <span class="moon">🌙</span>
            <span class="star star1">✨</span>
            <span class="star star2">⭐</span>
          </div>
        </div>
        <div class="header-text">
          <h1 class="title">
            <span class="title-main">雨声助眠</span>
            <span class="title-sub">Rain Sleep</span>
          </h1>
          <p class="description">
            <span class="raindrop">💧</span>
            听着雨声，安然入梦
            <span class="raindrop">💧</span>
          </p>
        </div>
        <div class="time-display" v-if="currentTime">
          <div class="time-card">
            <span class="time">{{ currentTime }}</span>
            <span class="date">{{ currentDate }}</span>
          </div>
        </div>
      </header>

      <!-- 定时器区域 -->
      <div class="timer-section">
        <div class="timer-header">
          <span class="timer-icon">⏰</span>
          <span class="timer-label">睡眠定时</span>
          <span class="timer-tip">倒计时结束自动关闭</span>
        </div>
        <div class="timer-buttons">
          <button v-for="min in timerOptions" :key="min"
            :class="['timer-btn', { active: selectedTimer === min && min > 0, 'cancel-btn': min === 0 }]"
            @click="setTimer(min)">
            {{ min === 0 ? '关闭定时' : min + '分钟' }}
          </button>
        </div>
        <div class="custom-timer">
          <input type="number" v-model="customMinutes" placeholder="自定义分钟" class="custom-input"
            @keyup.enter="setCustomTimer" />
          <button class="custom-set-btn" @click="setCustomTimer">设置</button>
        </div>
        <div class="timer-status" v-if="timerRemaining > 0">
          <div class="status-bar">
            <div class="status-fill" :style="{ width: timerProgress + '%' }"></div>
          </div>
          <div class="status-text">
            <span>⏳ 剩余时间</span>
            <span class="countdown">{{ formatTime(timerRemaining) }}</span>
          </div>
        </div>
      </div>

      <!-- 音频播放器组件 -->
      <AudioPlayer ref="audioPlayerRef" @playing="onAudioPlaying" @stopped="onAudioStopped" />

      <!-- 底部添加自定义音频 -->
      <div class="add-section">
        <div class="add-header">
          <span class="add-icon">🎵</span>
          <span class="add-label">添加自定义音频</span>
          <span class="add-hint">支持云存储链接</span>
        </div>
        <div class="add-form">
          <div class="form-group">
            <input type="text" v-model="newAudioTitle" placeholder="音频名称，如：森林小雨" class="form-input" maxlength="50" />
          </div>
          <div class="form-group">
            <input type="text" v-model="newAudioUrl" placeholder="音频URL（支持.mp3格式）" class="form-input" />
          </div>
          <button class="add-btn" @click="addCustomAudio">
            <span>➕</span> 添加到列表
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import AudioPlayer from './components/AudioPlayer.vue'

export default {
  name: 'App',
  components: {
    AudioPlayer
  },
  data() {
    return {
      // 定时器相关
      timerOptions: [5, 10, 15, 20, 30, 0],
      selectedTimer: null,
      customMinutes: '',
      timerRemaining: 0, // 剩余秒数
      timerInterval: null,
      timerEndTime: null,

      // 添加音频相关
      newAudioTitle: '',
      newAudioUrl: '',

      // 时间显示
      currentTime: '',
      currentDate: '',
      timeInterval: null
    }
  },
  computed: {
    // 定时器进度百分比
    timerProgress() {
      if (this.selectedTimer && this.selectedTimer > 0 && this.timerRemaining > 0) {
        const totalSeconds = this.selectedTimer * 60
        return (this.timerRemaining / totalSeconds) * 100
      }
      return 0
    }
  },
  mounted() {
    this.updateDateTime()
    this.timeInterval = setInterval(() => {
      this.updateDateTime()
    }, 1000)
  },
  beforeUnmount() {
    if (this.timeInterval) {
      clearInterval(this.timeInterval)
    }
    this.clearTimer()
  },
  methods: {
    // 更新时间显示
    updateDateTime() {
      const now = new Date()
      const hours = now.getHours().toString().padStart(2, '0')
      const minutes = now.getMinutes().toString().padStart(2, '0')
      this.currentTime = `${hours}:${minutes}`

      const month = now.getMonth() + 1
      const day = now.getDate()
      const weekdays = ['周日', '周一', '周二', '周三', '周四', '周五', '周六']
      this.currentDate = `${month}月${day}日 ${weekdays[now.getDay()]}`
    },

    // 设置定时器
    setTimer(minutes) {
      this.selectedTimer = minutes

      if (minutes === 0) {
        this.clearTimer()
        this.$refs.audioPlayerRef?.showToast('已关闭定时', 'info')
        return
      }

      // 清除旧定时器
      this.clearTimer()

      // 设置新定时器
      this.timerRemaining = minutes * 60
      this.timerEndTime = Date.now() + (minutes * 60 * 1000)

      this.timerInterval = setInterval(() => {
        const remaining = Math.max(0, Math.floor((this.timerEndTime - Date.now()) / 1000))
        this.timerRemaining = remaining

        if (remaining <= 0) {
          // 时间到，停止播放并退出APP
          this.clearTimer()
          this.$refs.audioPlayerRef?.stopPlayback()
          this.$refs.audioPlayerRef?.showToast('⏰ 定时结束，即将关闭', 'warning')

          // 延迟1秒后退出APP（给用户看到提示）
          setTimeout(() => {
            this.exitApp()
          }, 1000)
        }
      }, 1000)

      this.$refs.audioPlayerRef?.showToast(`已设置 ${minutes} 分钟后自动关闭`, 'success')
    },

    // 设置自定义定时
    setCustomTimer() {
      let minutes = parseInt(this.customMinutes)
      if (isNaN(minutes) || minutes <= 0) {
        this.$refs.audioPlayerRef?.showToast('请输入有效的分钟数（1-180）', 'error')
        return
      }
      minutes = Math.min(minutes, 180) // 最大3小时
      this.setTimer(minutes)
      this.customMinutes = ''
    },

    // 清除定时器
    clearTimer() {
      if (this.timerInterval) {
        clearInterval(this.timerInterval)
        this.timerInterval = null
      }
      this.timerRemaining = 0
      this.timerEndTime = null
    },

    // 格式化时间显示
    formatTime(seconds) {
      const mins = Math.floor(seconds / 60)
      const secs = seconds % 60
      return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`
    },

    // 退出APP（Capacitor环境）
    async exitApp() {
      // 检查是否在Capacitor环境中
      if (window.Capacitor && window.Capacitor.isNativePlatform()) {
        try {
          const { App } = await import('@capacitor/app')
          await App.exitApp()
        } catch (err) {
          console.log('退出APP失败', err)
          // 降级方案：停止所有音频
          this.$refs.audioPlayerRef?.stopPlayback()
        }
      } else {
        // Web环境：停止播放即可
        this.$refs.audioPlayerRef?.stopPlayback()
        this.$refs.audioPlayerRef?.showToast('Web环境：已停止播放（打包后支持自动退出）', 'info')
      }
    },

    // 添加自定义音频
    addCustomAudio() {
      if (!this.newAudioTitle.trim()) {
        this.$refs.audioPlayerRef?.showToast('请输入音频名称', 'error')
        return
      }
      if (!this.newAudioUrl.trim()) {
        this.$refs.audioPlayerRef?.showToast('请输入音频链接', 'error')
        return
      }

      // 简单URL校验
      const url = this.newAudioUrl.trim()
      if (!url.startsWith('http://') && !url.startsWith('https://')) {
        this.$refs.audioPlayerRef?.showToast('请输入有效的HTTP/HTTPS链接', 'error')
        return
      }

      // 添加到播放器
      const success = this.$refs.audioPlayerRef?.addAudio({
        title: this.newAudioTitle.trim(),
        url: url
      })

      if (success) {
        this.newAudioTitle = ''
        this.newAudioUrl = ''
      }
    },

    // 音频播放时的回调（重置定时器？可以选择不重置）
    onAudioPlaying() {
      // 可选：播放时如果有定时器，重新开始？暂不处理
    },

    onAudioStopped() {
      // 如果用户手动停止，且定时器还在，清除定时器
      if (this.timerRemaining > 0) {
        this.clearTimer()
        this.selectedTimer = null
      }
    }
  }
}
</script>

<style scoped>
.app-container {
  min-height: 100vh;
  position: relative;
  overflow-x: hidden;
  padding: 20px 16px 40px;
}

/* 动态渐变背景 */
.bg-gradient {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, #0a0a0f 0%, #0f0f1a 30%, #1a1a2e 70%, #0d0d1a 100%);
  z-index: -2;
}

/* 雨滴效果背景 */
.bg-rain {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: radial-gradient(circle at 10% 20%, rgba(100, 150, 200, 0.03) 2px, transparent 2px);
  background-size: 30px 30px;
  z-index: -1;
  pointer-events: none;
  animation: rainMove 20s linear infinite;
}

@keyframes rainMove {
  0% {
    background-position: 0 0;
  }

  100% {
    background-position: 0 30px;
  }
}

/* 头部样式 */
.header {
  position: relative;
  margin-bottom: 32px;
  padding: 20px 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 16px;
}

.header-glow {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 200px;
  height: 200px;
  background: radial-gradient(circle, rgba(100, 150, 255, 0.08) 0%, transparent 70%);
  filter: blur(40px);
  z-index: -1;
}

.moon-phase {
  flex-shrink: 0;
}

.moon-icon {
  position: relative;
  font-size: 48px;
  filter: drop-shadow(0 0 20px rgba(255, 255, 200, 0.3));
  animation: moonGlow 4s ease-in-out infinite;
}

@keyframes moonGlow {

  0%,
  100% {
    filter: drop-shadow(0 0 10px rgba(255, 255, 200, 0.3));
  }

  50% {
    filter: drop-shadow(0 0 25px rgba(255, 255, 200, 0.6));
  }
}

.star {
  position: absolute;
  font-size: 14px;
  opacity: 0.8;
  animation: twinkle 3s ease-in-out infinite;
}

.star1 {
  top: -10px;
  right: -10px;
  animation-delay: 0s;
}

.star2 {
  bottom: -5px;
  left: -5px;
  animation-delay: 1.5s;
  font-size: 10px;
}

@keyframes twinkle {

  0%,
  100% {
    opacity: 0.3;
    transform: scale(0.8);
  }

  50% {
    opacity: 1;
    transform: scale(1.2);
  }
}

.header-text {
  flex: 1;
}

.title {
  display: flex;
  flex-direction: column;
}

.title-main {
  font-size: 26px;
  font-weight: 700;
  background: linear-gradient(135deg, #e8e8ff 0%, #b8c4ff 50%, #9ab3ff 100%);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  letter-spacing: 2px;
}

.title-sub {
  font-size: 10px;
  letter-spacing: 3px;
  color: rgba(180, 200, 255, 0.5);
  margin-top: 4px;
}

.description {
  margin-top: 8px;
  font-size: 12px;
  color: rgba(180, 200, 255, 0.4);
  display: flex;
  align-items: center;
  gap: 6px;
}

.raindrop {
  animation: drop 2s ease-in-out infinite;
}

@keyframes drop {

  0%,
  100% {
    transform: translateY(0px);
    opacity: 0.5;
  }

  50% {
    transform: translateY(3px);
    opacity: 1;
  }
}

.time-display {
  flex-shrink: 0;
}

.time-card {
  background: rgba(30, 35, 60, 0.6);
  backdrop-filter: blur(10px);
  padding: 10px 16px;
  border-radius: 28px;
  text-align: center;
  border: 1px solid rgba(100, 150, 200, 0.2);
}

.time {
  font-size: 20px;
  font-weight: 600;
  color: #c8d8ff;
  letter-spacing: 1px;
}

.date {
  font-size: 10px;
  color: rgba(160, 180, 220, 0.6);
  margin-left: 8px;
}

/* 定时器区域 */
.timer-section {
  background: rgba(20, 25, 45, 0.5);
  backdrop-filter: blur(20px);
  border-radius: 28px;
  padding: 18px 20px;
  margin-bottom: 20px;
  border: 1px solid rgba(100, 150, 200, 0.15);
}

.timer-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.timer-icon {
  font-size: 18px;
}

.timer-label {
  font-size: 15px;
  font-weight: 500;
  color: #c8d8ff;
}

.timer-tip {
  font-size: 11px;
  color: rgba(160, 180, 220, 0.5);
  margin-left: auto;
}

.timer-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 16px;
}

.timer-btn {
  background: rgba(30, 35, 60, 0.8);
  border: 1px solid rgba(100, 150, 200, 0.2);
  padding: 10px 18px;
  border-radius: 30px;
  color: #b8c8f0;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.25s ease;
}

.timer-btn.active {
  background: linear-gradient(135deg, #4a6cff, #3a55cc);
  border-color: #6a8eff;
  color: white;
  box-shadow: 0 4px 15px rgba(74, 108, 255, 0.3);
}

.timer-btn.cancel-btn {
  background: rgba(80, 60, 70, 0.6);
  border-color: rgba(200, 100, 100, 0.3);
}

.custom-timer {
  display: flex;
  gap: 10px;
  margin-top: 12px;
}

.custom-input {
  flex: 1;
  background: rgba(30, 35, 60, 0.8);
  border: 1px solid rgba(100, 150, 200, 0.2);
  padding: 10px 14px;
  border-radius: 28px;
  color: #e0e8ff;
  font-size: 13px;
  outline: none;
}

.custom-input:focus {
  border-color: #6a8eff;
}

.custom-set-btn {
  background: rgba(74, 108, 255, 0.7);
  border: none;
  padding: 10px 20px;
  border-radius: 28px;
  color: white;
  font-size: 13px;
  cursor: pointer;
}

.timer-status {
  margin-top: 16px;
  padding-top: 12px;
  border-top: 1px solid rgba(100, 150, 200, 0.1);
}

.status-bar {
  height: 3px;
  background: rgba(100, 150, 200, 0.2);
  border-radius: 2px;
  overflow: hidden;
  margin-bottom: 8px;
}

.status-fill {
  height: 100%;
  background: linear-gradient(90deg, #6a8eff, #a0b8ff);
  border-radius: 2px;
  transition: width 0.3s linear;
}

.status-text {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: rgba(160, 180, 220, 0.6);
}

.countdown {
  color: #ffd966;
  font-weight: 500;
  font-family: monospace;
  font-size: 14px;
}

/* 添加音频区域 */
.add-section {
  background: rgba(20, 25, 45, 0.5);
  backdrop-filter: blur(20px);
  border-radius: 28px;
  padding: 18px 20px;
  margin-top: 20px;
  border: 1px solid rgba(100, 150, 200, 0.15);
}

.add-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.add-icon {
  font-size: 18px;
}

.add-label {
  font-size: 15px;
  font-weight: 500;
  color: #c8d8ff;
}

.add-hint {
  font-size: 11px;
  color: rgba(160, 180, 220, 0.5);
  margin-left: auto;
}

.add-form {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.form-group input {
  width: 100%;
  background: rgba(30, 35, 60, 0.8);
  border: 1px solid rgba(100, 150, 200, 0.2);
  padding: 12px 16px;
  border-radius: 24px;
  color: #e0e8ff;
  font-size: 14px;
  outline: none;
  transition: all 0.2s;
}

.form-group input:focus {
  border-color: #6a8eff;
  background: rgba(30, 35, 60, 0.95);
}

.form-group input::placeholder {
  color: rgba(160, 180, 220, 0.4);
}

.add-btn {
  background: linear-gradient(135deg, #3a55cc, #2a3fa0);
  border: none;
  padding: 12px;
  border-radius: 28px;
  color: white;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  transition: all 0.2s;
}

.add-btn:hover {
  transform: scale(0.98);
  background: linear-gradient(135deg, #4a6cff, #3a55cc);
}

/* 响应式 */
@media (max-width: 500px) {
  .app-container {
    padding: 16px 12px 32px;
  }

  .timer-buttons {
    gap: 8px;
  }

  .timer-btn {
    padding: 8px 14px;
    font-size: 12px;
  }

  .title-main {
    font-size: 22px;
  }

  .moon-icon {
    font-size: 40px;
  }
}
</style>