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
                            {{ currentAudio && currentAudio.id === audio.id && isPlaying ? '🌧️' : '🎵' }}
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
// 默认音频列表（使用和HTML一样的可用的在线音频）
import { DEFAULT_AUDIOS } from './audios'
// 存储key
const STORAGE_KEY = 'rainsleep_audio_list'

export default {
    name: 'AudioPlayer',
    data() {
        return {
            audioList: [],
            currentAudio: null,
            currentAudioElement: null,
            isPlaying: false,
            toastMessage: '',
            toastType: 'info',
            toastTimer: null,
            DEFAULT_AUDIOS
        }
    },
    mounted() {
        this.loadAudioList()
    },
    beforeUnmount() {
        if (this.currentAudioElement) {
            this.currentAudioElement.pause()
            this.currentAudioElement = null
        }
        if (this.toastTimer) {
            clearTimeout(this.toastTimer)
        }
    },
    methods: {
        // 加载音频列表（从localStorage）
        loadAudioList() {
            const saved = localStorage.getItem(STORAGE_KEY)
            if (saved) {
                try {
                    const parsed = JSON.parse(saved)
                    // 合并默认音频和自定义音频，去重
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

        // 保存到localStorage
        saveAudioList() {
            const toSave = this.audioList.filter(a => !a.isDefault)
            localStorage.setItem(STORAGE_KEY, JSON.stringify(toSave))
        },

        // 播放音频（使用和HTML完全一样的逻辑）
        playAudio(audio) {
            // 如果正在播放同一首，停止它（HTML版本是重新开始）
            if (this.currentAudio && this.currentAudio.id === audio.id && this.currentAudioElement) {
                this.stopPlayback()
            }

            // 停止当前播放
            if (this.currentAudioElement) {
                this.currentAudioElement.pause()
                this.currentAudioElement = null
            }

            try {
                // 创建新的音频对象（和HTML完全一样）
                const audioObj = new Audio()
                audioObj.src = audio.url
                audioObj.loop = true // 循环播放
                audioObj.crossOrigin = 'anonymous' // 允许跨域

                // 播放（和HTML一样的逻辑）
                const playPromise = audioObj.play()
                if (playPromise !== undefined) {
                    playPromise.then(() => {
                        this.currentAudioElement = audioObj
                        this.currentAudio = audio
                        this.isPlaying = true
                        this.showToast(`🎵 正在播放：${audio.title}`, 'success')
                    }).catch(err => {
                        console.error('播放失败:', err)
                        this.showToast('播放失败，请检查音频链接是否有效', 'error')
                    })
                }

                // 音频结束时自动重播（和HTML一样）
                audioObj.addEventListener('ended', () => {
                    if (this.currentAudioElement === audioObj) {
                        audioObj.currentTime = 0
                        audioObj.play().catch(e => console.log('重播失败', e))
                    }
                })

            } catch (err) {
                console.error('创建音频失败:', err)
                this.showToast('无法播放该音频', 'error')
            }
        },

        // 暂停
        pauseAudio() {
            if (this.currentAudioElement && this.isPlaying) {
                this.currentAudioElement.pause()
                this.isPlaying = false
                this.showToast('已暂停', 'info')
            }
        },

        // 恢复播放
        resumeAudio() {
            if (this.currentAudioElement && !this.isPlaying) {
                const playPromise = this.currentAudioElement.play()
                if (playPromise !== undefined) {
                    playPromise.then(() => {
                        this.isPlaying = true
                        this.showToast('继续播放', 'info')
                    }).catch(err => {
                        console.error('恢复播放失败', err)
                    })
                }
            }
        },

        // 停止播放
        stopPlayback() {
            if (this.currentAudioElement) {
                this.currentAudioElement.pause()
                this.currentAudioElement = null
            }
            this.currentAudio = null
            this.isPlaying = false
            this.$emit('stopped')
        },

        // 添加自定义音频
        addAudio(audioData) {
            // 检查是否已存在相同URL
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

            // 如果正在播放，先停止
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
.audio-player {
    position: relative;
}

/* 当前播放卡片 */
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

/* 音频列表容器 */
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

/* 音频列表 - 支持滚动 */
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

/* Toast 提示 */
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