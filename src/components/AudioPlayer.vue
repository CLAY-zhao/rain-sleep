<!-- src/components/AudioPlayer.vue -->
<template>
    <div class="audio-player">
        <!-- 当前播放信息 -->
        <div class="current-playing" v-if="currentAudio">
            <div class="playing-card">
                <div class="playing-icon">
                    <span class="playing-animation" v-if="isPlaying">
                        <span class="wave-bar"></span>
                        <span class="wave-bar"></span>
                        <span class="wave-bar"></span>
                        <span class="wave-bar"></span>
                        <span class="wave-bar"></span>
                    </span>
                    <span v-else class="pause-icon">🎵</span>
                </div>
                <div class="playing-info">
                    <div class="playing-title">{{ currentAudio.title }}</div>
                    <div class="playing-status">{{ isPlaying ? '播放中' : '已暂停' }}</div>
                </div>
                <button class="stop-playing-btn" @click="stopPlayback" v-if="isPlaying">
                    ⏹️
                </button>
            </div>
        </div>

        <!-- 音频列表 -->
        <div class="audio-list-container">
            <div class="list-header">
                <span class="list-icon">🎧</span>
                <span class="list-title">催眠音频库</span>
                <span class="list-count">{{ audioList.length }}首</span>
            </div>

            <div class="audio-list" :class="{ 'has-scroll': audioList.length > 5 }">
                <div v-for="audio in audioList" :key="audio.id"
                    :class="['audio-list-item', { active: currentAudio && currentAudio.id === audio.id }]"
                    @click="playAudio(audio)">
                    <div class="item-info">
                        <div class="item-icon">
                            {{ getAudioIcon(audio.title) }}
                        </div>
                        <div class="item-details">
                            <div class="item-title">{{ audio.title }}</div>
                            <div class="item-duration">{{ audio.duration || '自定义' }}</div>
                        </div>
                    </div>
                    <div class="item-action">
                        <button v-if="!audio.isDefault" class="delete-btn" @click.stop="deleteAudio(audio.id)">
                            🗑️
                        </button>
                        <div class="play-indicator" v-if="currentAudio && currentAudio.id === audio.id && isPlaying">
                            <span></span><span></span><span></span>
                        </div>
                    </div>
                </div>

                <div v-if="audioList.length === 0" class="empty-list">
                    <span>🎵</span>
                    <p>暂无音频，请在下方添加</p>
                </div>
            </div>
        </div>

        <!-- Toast 提示 -->
        <transition name="toast-fade">
            <div v-if="toastMessage" class="toast-message" :class="toastType">
                {{ toastMessage }}
            </div>
        </transition>
    </div>
</template>

<script>
// 默认音频列表
import { DEFAULT_AUDIOS } from './audios'
// 存储key
const STORAGE_KEY = 'rainsleep_audio_list'

export default {
    name: 'AudioPlayer',
    data() {
        return {
            audioList: [],
            currentAudio: null,
            audioContext: null,        // Web Audio API 上下文
            audioBuffer: null,          // 缓存的音频数据
            sourceNode: null,           // 当前播放源
            gainNode: null,             // 音量控制节点
            isPlaying: false,
            toastMessage: '',
            toastType: 'info',
            toastTimer: null,
            fadeInterval: null,         // 淡入淡出定时器
            isFading: false,              // 是否正在淡变
            DEFAULT_AUDIOS
        }
    },
    mounted() {
        this.loadAudioList()
        // 初始化 Web Audio API
        this.initAudioContext()
    },
    beforeUnmount() {
        if (this.toastTimer) {
            clearTimeout(this.toastTimer)
        }
        if (this.fadeInterval) {
            clearInterval(this.fadeInterval)
        }
        if (this.audioContext) {
            this.audioContext.close()
        }
    },
    methods: {
        // 获取音频图标
        getAudioIcon(title) {
            if (title.includes('沙漠之夜')) return '🏜️'
            if (title.includes('旷野蟋蟀')) return '🦗'
            if (title.includes('夜晚蛐蛐')) return '🌙'
            if (title.includes('溪流与虫鸣')) return '🏞️'
            if (title.includes('轻柔小雨声')) return '🌧️'
            if (title.includes('雷雨')) return '🌩️'
            return '🎵'
        },

        // 初始化 Web Audio API
        initAudioContext() {
            try {
                // 创建 AudioContext（需要用户交互后才能启动）
                this.audioContext = new (window.AudioContext || window.webkitAudioContext)()
                // 创建增益节点（控制音量）
                this.gainNode = this.audioContext.createGain()
                this.gainNode.connect(this.audioContext.destination)
                this.gainNode.gain.value = 0.8 // 设置初始音量
            } catch (err) {
                console.error('Web Audio API 初始化失败:', err)
            }
        },

        // 加载并缓存音频
        async loadAudioBuffer(url) {
            try {
                const response = await fetch(url)
                const arrayBuffer = await response.arrayBuffer()
                const audioBuffer = await this.audioContext.decodeAudioData(arrayBuffer)
                return audioBuffer
            } catch (err) {
                console.error('加载音频失败:', err)
                throw err
            }
        },

        // 播放音频（无缝循环）
        async playAudio(audio) {
            // 如果正在播放同一首，停止
            if (this.currentAudio && this.currentAudio.id === audio.id && this.isPlaying) {
                this.pauseAudio()
                return
            }

            // 停止当前播放
            this.stopPlayback()

            // 恢复 AudioContext（如果被挂起）
            if (this.audioContext && this.audioContext.state === 'suspended') {
                await this.audioContext.resume()
            }

            this.showToast(`🎵 正在加载：${audio.title}`, 'info')

            try {
                // 加载音频
                const buffer = await this.loadAudioBuffer(audio.url)
                this.audioBuffer = buffer
                this.currentAudio = audio

                // 开始播放（带淡入效果）
                this.startSeamlessLoop()

            } catch (err) {
                console.error('播放失败:', err)
                this.showToast('播放失败，请检查音频链接', 'error')
            }
        },

        // 开始无缝循环播放
        startSeamlessLoop() {
            if (!this.audioBuffer || !this.audioContext) return

            // 淡出当前播放（如果有）
            if (this.sourceNode && this.isPlaying) {
                this.fadeOut(() => {
                    this.createAndPlaySource()
                })
            } else {
                this.createAndPlaySource()
            }
        },

        // 创建并播放音频源
        createAndPlaySource() {
            // 创建新的音频源
            const source = this.audioContext.createBufferSource()
            source.buffer = this.audioBuffer
            source.loop = true  // 启用循环

            // 连接节点：source -> gain -> destination
            source.connect(this.gainNode)

            // 保存当前源
            this.sourceNode = source

            // 开始播放（从开头）
            source.start(0)
            this.isPlaying = true

            // 淡入
            this.fadeIn()

            // 监听播放结束（理论上 loop=true 不会触发，但以防万一）
            source.onended = () => {
                if (this.sourceNode === source && this.isPlaying) {
                    // 如果循环失效，重新创建
                    this.createAndPlaySource()
                }
            }

            this.$emit('playing')
            this.showToast(`🎵 正在播放：${this.currentAudio.title}`, 'success')
        },

        // 淡入效果（0.5秒）
        fadeIn() {
            if (this.fadeInterval) clearInterval(this.fadeInterval)
            this.isFading = true

            const targetVolume = 0.8
            const startVolume = this.gainNode.gain.value
            const duration = 500 // 毫秒
            const steps = 20
            const stepTime = duration / steps
            const stepValue = (targetVolume - startVolume) / steps

            let currentStep = 0

            this.fadeInterval = setInterval(() => {
                currentStep++
                const newVolume = startVolume + (stepValue * currentStep)
                this.gainNode.gain.value = Math.min(targetVolume, Math.max(0, newVolume))

                if (currentStep >= steps) {
                    clearInterval(this.fadeInterval)
                    this.fadeInterval = null
                    this.isFading = false
                }
            }, stepTime)
        },

        // 淡出效果（0.5秒）
        fadeOut(callback) {
            if (this.fadeInterval) clearInterval(this.fadeInterval)
            this.isFading = true

            const startVolume = this.gainNode.gain.value
            const duration = 500
            const steps = 20
            const stepTime = duration / steps
            const stepValue = startVolume / steps

            let currentStep = 0

            this.fadeInterval = setInterval(() => {
                currentStep++
                const newVolume = startVolume - (stepValue * currentStep)
                this.gainNode.gain.value = Math.max(0, newVolume)

                if (currentStep >= steps) {
                    clearInterval(this.fadeInterval)
                    this.fadeInterval = null
                    this.isFading = false

                    // 停止当前源
                    if (this.sourceNode) {
                        try {
                            this.sourceNode.stop()
                        } catch (e) {
                            return
                        }
                        this.sourceNode = null
                    }

                    if (callback) callback()
                }
            }, stepTime)
        },

        // 暂停音频（淡出后暂停）
        pauseAudio() {
            if (!this.isPlaying) return

            this.fadeOut(() => {
                if (this.sourceNode) {
                    try {
                        this.sourceNode.stop()
                    } catch (e) {
                        return
                    }
                    this.sourceNode = null
                }
                this.isPlaying = false
                this.showToast('已暂停', 'info')
            })
        },

        // 恢复播放（淡入）
        async resumeAudio() {
            if (this.isPlaying || !this.currentAudio) return

            // 恢复 AudioContext
            if (this.audioContext && this.audioContext.state === 'suspended') {
                await this.audioContext.resume()
            }

            // 重新创建播放源
            this.createAndPlaySource()
        },

        // 停止播放
        stopPlayback() {
            if (this.fadeInterval) {
                clearInterval(this.fadeInterval)
                this.fadeInterval = null
            }

            if (this.sourceNode) {
                try {
                    this.sourceNode.stop()
                } catch (e) {
                    return
                }
                this.sourceNode = null
            }

            this.isPlaying = false
            this.currentAudio = null
            this.isFading = false

            // 重置音量
            if (this.gainNode) {
                this.gainNode.gain.value = 0.8
            }

            this.$emit('stopped')
        },

        // 加载音频列表
        loadAudioList() {
            const saved = localStorage.getItem(STORAGE_KEY)
            if (saved) {
                try {
                    const parsed = JSON.parse(saved)
                    const defaultIds = DEFAULT_AUDIOS.map(d => d.id)
                    const customAudios = parsed.filter(a => !defaultIds.includes(a.id))
                    this.audioList = [...DEFAULT_AUDIOS, ...customAudios]
                } catch (e) {
                    this.audioList = [...DEFAULT_AUDIOS]
                }
            } else {
                this.audioList = [...DEFAULT_AUDIOS]
            }
            this.saveAudioList()
        },

        // 保存音频列表
        saveAudioList() {
            const toSave = this.audioList.filter(a => !a.isDefault)
            localStorage.setItem(STORAGE_KEY, JSON.stringify(toSave))
        },

        // 添加自定义音频
        addAudio(audioData) {
            const exists = this.audioList.some(a => a.url === audioData.url)
            if (exists) {
                this.showToast('该音频链接已存在', 'error')
                return false
            }

            const newId = Math.max(...this.audioList.map(a => a.id), 0) + 1
            const newAudio = {
                id: newId,
                title: audioData.title,
                url: audioData.url,
                duration: '自定义',
                isDefault: false
            }

            this.audioList.push(newAudio)
            this.saveAudioList()
            this.showToast(`✅ 已添加：${audioData.title}`, 'success')
            return true
        },

        // 删除自定义音频
        deleteAudio(id) {
            const audio = this.audioList.find(a => a.id === id)
            if (audio && audio.isDefault) {
                this.showToast('默认音频不可删除', 'error')
                return
            }

            if (this.currentAudio && this.currentAudio.id === id) {
                this.stopPlayback()
            }

            this.audioList = this.audioList.filter(a => a.id !== id)
            this.saveAudioList()
            this.showToast('已删除', 'info')
        },

        // 显示提示
        showToast(message, type = 'info') {
            if (this.toastTimer) {
                clearTimeout(this.toastTimer)
            }
            this.toastMessage = message
            this.toastType = type
            this.toastTimer = setTimeout(() => {
                this.toastMessage = ''
            }, 2000)
        }
    }
}
</script>

<style scoped>
/* 样式保持不变，和之前一样 */
.audio-player {
    position: relative;
}

.current-playing {
    margin-bottom: 20px;
}

.playing-card {
    background: linear-gradient(135deg, rgba(40, 45, 75, 0.8), rgba(25, 30, 55, 0.9));
    backdrop-filter: blur(20px);
    border-radius: 60px;
    padding: 12px 20px;
    display: flex;
    align-items: center;
    gap: 16px;
    border: 1px solid rgba(100, 150, 200, 0.25);
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
}

.playing-icon {
    width: 48px;
    height: 48px;
    background: rgba(74, 108, 255, 0.15);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 24px;
}

.playing-animation {
    display: flex;
    align-items: center;
    gap: 3px;
    height: 30px;
}

.wave-bar {
    width: 3px;
    height: 15px;
    background: linear-gradient(180deg, #6a8eff, #a0b8ff);
    border-radius: 2px;
    animation: wavePlay 0.8s ease-in-out infinite;
}

.wave-bar:nth-child(1) {
    animation-delay: 0s;
    height: 12px;
}

.wave-bar:nth-child(2) {
    animation-delay: 0.15s;
    height: 20px;
}

.wave-bar:nth-child(3) {
    animation-delay: 0.3s;
    height: 28px;
}

.wave-bar:nth-child(4) {
    animation-delay: 0.45s;
    height: 20px;
}

.wave-bar:nth-child(5) {
    animation-delay: 0.6s;
    height: 12px;
}

@keyframes wavePlay {

    0%,
    100% {
        transform: scaleY(0.6);
        opacity: 0.5;
    }

    50% {
        transform: scaleY(1);
        opacity: 1;
    }
}

.pause-icon {
    font-size: 28px;
    opacity: 0.7;
}

.playing-info {
    flex: 1;
}

.playing-title {
    font-size: 16px;
    font-weight: 600;
    color: #e0e8ff;
    margin-bottom: 4px;
}

.playing-status {
    font-size: 11px;
    color: rgba(160, 180, 220, 0.6);
}

.stop-playing-btn {
    background: rgba(200, 80, 80, 0.2);
    border: none;
    width: 36px;
    height: 36px;
    border-radius: 50%;
    font-size: 18px;
    cursor: pointer;
    transition: all 0.2s;
}

.stop-playing-btn:hover {
    background: rgba(200, 80, 80, 0.5);
}

.audio-list-container {
    background: rgba(20, 25, 45, 0.5);
    backdrop-filter: blur(20px);
    border-radius: 28px;
    overflow: hidden;
    border: 1px solid rgba(100, 150, 200, 0.15);
}

.list-header {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 16px 20px;
    border-bottom: 1px solid rgba(100, 150, 200, 0.1);
}

.list-icon {
    font-size: 16px;
}

.list-title {
    font-size: 15px;
    font-weight: 500;
    color: #c8d8ff;
}

.list-count {
    margin-left: auto;
    font-size: 11px;
    color: rgba(160, 180, 220, 0.5);
    background: rgba(100, 150, 200, 0.1);
    padding: 4px 8px;
    border-radius: 20px;
}

.audio-list {
    max-height: 380px;
    overflow-y: auto;
}

.audio-list {
    scrollbar-width: none;
    -ms-overflow-style: none;
}

.audio-list::-webkit-scrollbar {
    display: none;
}

.audio-list-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 14px 20px;
    border-bottom: 1px solid rgba(100, 150, 200, 0.05);
    cursor: pointer;
    transition: all 0.2s;
}

.audio-list-item:hover {
    background: rgba(74, 108, 255, 0.1);
}

.audio-list-item.active {
    background: linear-gradient(90deg, rgba(74, 108, 255, 0.15), transparent);
    border-left: 3px solid #6a8eff;
}

.item-info {
    display: flex;
    align-items: center;
    gap: 12px;
    flex: 1;
}

.item-icon {
    width: 32px;
    font-size: 18px;
    text-align: center;
}

.item-details {
    flex: 1;
}

.item-title {
    font-size: 14px;
    font-weight: 500;
    color: #d0d8ff;
    margin-bottom: 4px;
}

.item-duration {
    font-size: 11px;
    color: rgba(160, 180, 220, 0.5);
}

.item-action {
    display: flex;
    align-items: center;
    gap: 12px;
}

.delete-btn {
    background: transparent;
    border: none;
    font-size: 16px;
    cursor: pointer;
    opacity: 0.5;
    transition: opacity 0.2s;
    padding: 6px;
}

.delete-btn:hover {
    opacity: 1;
}

.play-indicator {
    display: flex;
    gap: 3px;
    align-items: center;
}

.play-indicator span {
    width: 3px;
    height: 10px;
    background: #6a8eff;
    border-radius: 2px;
    animation: waveSmall 0.8s ease-in-out infinite;
}

.play-indicator span:nth-child(1) {
    animation-delay: 0s;
}

.play-indicator span:nth-child(2) {
    animation-delay: 0.2s;
}

.play-indicator span:nth-child(3) {
    animation-delay: 0.4s;
}

@keyframes waveSmall {

    0%,
    100% {
        height: 6px;
        opacity: 0.5;
    }

    50% {
        height: 14px;
        opacity: 1;
    }
}

.empty-list {
    text-align: center;
    padding: 48px 20px;
    color: rgba(160, 180, 220, 0.4);
}

.empty-list span {
    font-size: 48px;
    opacity: 0.5;
    display: block;
    margin-bottom: 12px;
}

.empty-list p {
    font-size: 13px;
}

.toast-message {
    position: fixed;
    bottom: 100px;
    left: 50%;
    transform: translateX(-50%);
    padding: 12px 24px;
    border-radius: 40px;
    font-size: 13px;
    font-weight: 500;
    z-index: 2000;
    backdrop-filter: blur(20px);
    white-space: nowrap;
    max-width: 80vw;
    white-space: normal;
    word-break: break-word;
    text-align: center;
}

.toast-message.success {
    background: rgba(30, 80, 50, 0.9);
    color: #a0ffc0;
    border: 1px solid rgba(80, 200, 120, 0.3);
}

.toast-message.error {
    background: rgba(100, 40, 40, 0.9);
    color: #ffb0b0;
    border: 1px solid rgba(255, 80, 80, 0.3);
}

.toast-message.info {
    background: rgba(40, 45, 70, 0.95);
    color: #c0d0ff;
    border: 1px solid rgba(100, 150, 200, 0.3);
}

.toast-message.warning {
    background: rgba(100, 70, 30, 0.9);
    color: #ffe0a0;
    border: 1px solid rgba(255, 180, 50, 0.3);
}

.toast-fade-enter-active,
.toast-fade-leave-active {
    transition: all 0.3s ease;
}

.toast-fade-enter-from,
.toast-fade-leave-to {
    opacity: 0;
    transform: translateX(-50%) translateY(20px);
}
</style>