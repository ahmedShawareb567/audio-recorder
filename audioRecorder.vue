<template lang="pug">
.record
    .record-button(:class="[{'is-active': active}]")
        p(v-if="active" @click="stop()"): svg-icon(name="micFill")
        p(v-else @click="record()"): svg-icon(name="micNone")
    .record-voice(:class="{'is-active': displayVoice}")
      button(:disabled="initialDisabledValue" @click="playAudio()" :class="{'is-active': displayWave}")
        span: svg-icon(name="play")
        .wave
          p
          p
          p
    .record-timer(ref="recordTimer")
        p {{this.time}}
        .timerProgress(ref="timerProgress" :class="{'is-active': progressStatus}")
    button.record-delete(:class="{'is-active': audioStop}")
      svg-icon(name="bin" @click="deleteAudio")
    audio(ref="audio" controls style="display: none")
      | audio is not supported in your browser
</template>
<script>
export default {
  name: 'audioRecorder',
  props: {
    resetAudio: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      active: false,
      recorder: null,
      chunks: null,
      stream: null,
      audioFile: null,
      audioStop: false,
      audioSize: 0,
      progressStatus: false,
      currentAudioLink: null,
      displayVoice: false,
      initialDisabledValue: true,
      displayWave: false,
      currentAudio: null,
      mediaOptions: {
        audio: {
          tag: 'audio',
          type: 'audio/mpeg',
          mimeType: 'audio/mpeg',
          ext: '.mp3',
          audioBitsPerSecond: '128000',
          gUM: { audio: true },
        },
      },
      time: '00:00:00',
      timeInterval: null,
      counter: 1,
      audioProps: {
        audioBitsPerSecond: 32000,
        mimeType: 'audio/mp4;codecs=acc',
        tag: 'audio',
        type: 'audio/mp4',
        ext: '.mp4',
      },
    }
  },
  mounted() {
    this.$refs.audio.addEventListener('ended', () => {
      this.displayWave = false
    })
  },
  methods: {
    record() {
      // RESET AUDIO
      this.$emit('audioStartRecord', {
        start: true,
      })
      this.active = true
      this.initialDisabledValue = true
      if (this.displayVoice) {
        this.$refs.audio.pause()
      }
      this.progressStatus = false
      this.displayVoice = false
      this.audioStop = false
      this.displayWave = false
      this.counter = 1
      this.currentAudioLink = null
      this.chunks = null
      this.audioFile = null
      this.$emit('audioValue', {
        audio: null,
        size: 0,
        sizeExt: 'MB',
        audioURL: null,
      })
      navigator.mediaDevices
        .getUserMedia({
          audio: true,
        })
        .then((_stream) => {
          this.stream = _stream
          this.recorder = new MediaRecorder(this.stream)
          this.recorder.start()
          // TIME INTERVAL
          this.timeInterval = setInterval(() => {
            this.timeConvert(this.counter)
            this.counter++
          }, 1000)
          this.recorder.addEventListener('dataavailable', (e) => {
            if (this.recorder.state == 'inactive') this.getAudioURL(e)
          })
        })
        .catch((er) => {
          console.log(er)
          this.active = false
          this.$emit('audioStartRecord', {
            start: false,
          })
        })
    },
    stop() {
      this.initialDisabledValue = false
      this.audioStop = true
      this.active = false
      this.displayVoice = true
      clearInterval(this.timeInterval)
      this.recorder.stop()
      this.progressStatus = true
      this.$emit('audioStartRecord', {
        start: false,
      })
    },
    getAudioURL(e) {
      this.chunks = e.data
      this.currentAudioLink = URL.createObjectURL(this.chunks)
      this.$refs.audio.src = this.currentAudioLink
      this.audioSize = (this.chunks.size / 1024 / 1024).toFixed(2)
      const blob = new Blob([this.chunks], this.audioProps)
      this.audioFile = new File([blob], `${Date.now()}-record.mp3`, {
        type: 'audio/mp3',
      })
      // EMIT VALUES :)
      this.$emit('audioValue', {
        audio: this.audioFile,
        size: this.audioSize,
        sizeExt: 'MB',
        audioURL: this.currentAudioLink,
      })
    },
    playAudio() {
      if (this.displayWave) {
        this.displayWave = false
        this.$refs.audio.pause()
        return
      }
      this.displayWave = true
      this.$refs.audio.play()
    },
    deleteAudio() {
      // EMIT VALUES :)
      if (this.displayWave) {
        this.displayWave = false
        this.$refs.audio.pause()
      }
      this.progressStatus = false
      this.initialDisabledValue = true
      this.audioStop = false
      this.time = '00:00:00'
      this.chunks = null
      this.$emit('audioValue', {
        audio: null,
        size: 0,
        sizeExt: 'MB',
        audioURL: 0,
      })
    },
    timeConvert(seconds) {
      const format = (val) => `0${Math.floor(val)}`.slice(-2)
      const hours = seconds / 3600
      const minutes = (seconds % 3600) / 60
      this.time = [hours, minutes, seconds % 60].map(format).join(':')
    },
  },
  watch: {
    resetAudio(newValue, oldValue) {
      if (newValue) {
        if (this.displayWave) {
          this.displayWave = false
          this.$refs.audio.pause()
        }
        this.initialDisabledValue = true
        this.audioStop = false
        this.time = '00:00:00'
        this.chunks = []
        this.$emit('audioValue', {
          audio: null,
          size: 0,
          sizeExt: 'MB',
          audioURL: 0,
        })
      }
    },
  },
}
</script>
<style lang="scss">
.record {
  display: flex;
  justify-content: center;
  align-items: center;
  &-button {
    position: relative;
    width: 3rem;
    height: 3rem;
    border-radius: 50%;
    background-color: #fff;
    overflow: hidden;
    margin: 0.25rem;
    box-shadow: 0rem 0.2rem 1rem 0rem rgba(0, 0, 0, 0.1);
    p {
      display: flex;
      justify-content: center;
      align-items: center;
      position: relative;
      outline: none;
      border: none;
      z-index: 2;
      cursor: pointer;
      width: 100%;
      height: 100%;
      background-color: transparent;
      margin: 0rem;
      border-radius: 50%;
      svg {
        width: 1.5rem;
        height: 1.5rem;
      }
    }
    &::after,
    &::before {
      content: '';
      position: absolute;
      top: 0rem;
      left: 0rem;
      z-index: 1;
      width: 100%;
      height: 100%;
      border-radius: 50%;
      transform: scale(0);
      border: 0.1rem solid #bc1d8e;
      transition: all 0.2s ease-in-out;
    }
    &.is-active {
      p {
        svg {
          fill: #bc1d8e;
        }
      }
      &::after {
        animation: scaleAnimation 1s linear infinite;
      }
      &::before {
        opacity: 0.1;
        background: #bc1d8e;
        animation: scaleAnimation 1s linear infinite;
        animation-delay: 0.5s;
      }
    }
  }
  &-voice {
    width: 2.5rem;
    height: 2.5rem;
    border-radius: 50%;
    background-color: #fff;
    margin: 0.25rem;
    box-shadow: 0rem 0.2rem 1rem 0rem rgba(0, 0, 0, 0.1);
    button {
      position: relative;
      display: flex;
      justify-content: center;
      align-items: center;
      position: relative;
      outline: none;
      border: none;
      z-index: 2;
      cursor: pointer;
      width: 100%;
      height: 100%;
      background-color: transparent;
      margin: 0rem;
      border-radius: 50%;
      span {
        svg {
          width: 1.1rem;
          height: 1.1rem;
        }
      }
      .wave {
        position: absolute;
        top: 0rem;
        left: 0rem;
        display: flex;
        align-items: center;
        justify-content: center;
        width: 100%;
        height: 100%;
        opacity: 0;
        z-index: 0;
        p {
          position: relative;
          margin: 0rem 0.1rem;
          width: 0.15rem;
          height: 1rem;
          background-color: #bc1d8e;
          border-radius: 0.5rem;
          animation: scaleAnimation 0.8s ease-in-out infinite;
          &:first-child {
            animation-delay: 0.1s;
          }
          &:nth-child(2) {
            animation-delay: 0.2s;
          }
          &:last-child {
            animation-delay: 0.3s;
          }
        }
      }
      &.is-active {
        span {
          opacity: 0 !important;
        }
        .wave {
          opacity: 1;
          z-index: 2;
        }
      }
    }
    &.is-active {
      button {
        span {
          opacity: 1;
        }
      }
    }
  }
  &-timer {
    position: relative;
    width: 2.8rem;
    height: 1rem;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
    padding: 0rem;
    margin: 0rem 0.2rem;
    background-color: rgba(0, 0, 0, 0.05);
    border-radius: 1rem;
    p {
      position: relative;
      z-index: 2;
      display: block;
      font-size: 0.6rem;
      padding: 0rem;
      margin: 0rem;
    }
    .timerProgress {
      position: absolute;
      top: 0rem;
      right: 0rem;
      width: 100%;
      height: 100%;
      z-index: 1;
      opacity: 0;
      background-color: #ffc4ee;
      &.is-active {
        opacity: 1;
      }
    }
  }
  &-delete {
    display: flex;
    justify-content: center;
    align-items: center;
    flex-shrink: 0;
    width: 2rem;
    height: 2rem;
    background-color: #ffc4ee;
    border-radius: 50%;
    outline: none !important;
    border: none;
    color: #bc1d8e;
    transform: scale(0);
    padding: 0rem;
    transition: all 0.2s ease-in-out;
    svg {
      display: block;
      width: 1.2rem;
      padding: 0rem;
      margin: 0rem;
      height: 1.2rem;
    }
  }
  .record-delete {
    &.is-active {
      transform: scale(1);
    }
  }
}
// @media (max-width: 767px) {
//   .record {
//     display: block;
//   }
// }
@keyframes scaleAnimation {
  0%,
  100% {
    transform: scale(0);
  }
  65% {
    transform: scale(1);
  }
}
</style>
