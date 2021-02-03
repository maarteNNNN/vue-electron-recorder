<template lang="pug">
  .container
    .row
      a.button(
        @click="record"
        :class="`shadow-mic-${analyserPercentage}`"
        )
        svg-icon(
          type="mdi"
          :path="mdiMicrophone"
          :size="50"
          :style="`color: ${recording ? 'red' : 'darkslategrey'}`"
          )
    .row
      p {{ timer || '00:00:00:000' }}
    .row
      p(v-if="error" style="color: red") {{ error.message }}
      p(v-if="message") File saved: {{ message && message.filePath }}
</template>

<script>
import RecordRTC from 'recordrtc'
import SvgIcon from '@jamescoyle/vue-icon'
import { mdiMicrophone } from '@mdi/js'
import { ipcRenderer } from 'electron'

export default {
  name: 'Recorder',
  data() {
    return {
      mdiMicrophone,
      recording: false,
      recorder: null,
      stream: null,
      timer: null,
      analyserPercentage: null,
      timerInterval: null,
      analyserInterval: null,
      error: null,
      message: null,
    }
  },
  mounted() {
    ipcRenderer.on('file-save', (event, { error, message }) => {
      this.message = null
      this.error = null
      if (error) {
        this.error = error
      } else {
        this.message = message
      }
    })
  },
  methods: {
    async record() {
      if (this.recording) {
        this.recording = false
        this.recorder.stopRecording(() => {
          this.stream.getTracks().forEach(track => {
            track.stop()
          })

          this.recorder.getDataURL(async dataURL => {
            const file = {
              audio: {
                type: this.recorder.getBlob().type || 'audio/wav',
                dataURL,
              },
            }

            ipcRenderer.send('file-save', file)
          })

          clearInterval(this.analyserInterval)
          clearInterval(this.timerInterval)
          this.analyserPercentage = 0
        })

        return
      }

      this.stream = await navigator.mediaDevices.getUserMedia({
        audio: { echoCancellation: true },
      })

      this.analyser()
      this.startTime = new Date()
      this.timerFn()

      this.recorder = RecordRTC(this.stream, {
        type: 'audio',

        mimeType: 'audio/webm',
        sampleRate: 44100,
        // used by StereoAudioRecorder
        // the range 22050 to 96000.
        // let us force 16khz recording:
        desiredSampRate: 16000,

        numberOfAudioChannels: 1,

        timeSlice: 1000,
      })

      this.recorder.startRecording()
      this.recording = true
    },
    analyser() {
      const audioContext = new AudioContext()
      const audioSource = audioContext.createMediaStreamSource(this.stream)
      const analyser = audioContext.createAnalyser()
      analyser.fftSize = 512
      analyser.minDecibels = -127
      analyser.maxDecibels = 0
      analyser.smoothingTimeConstant = 0.4
      audioSource.connect(analyser)
      const volumes = new Uint8Array(analyser.frequencyBinCount)
      this.analyserInterval = setInterval(() => {
        analyser.getByteFrequencyData(volumes)
        let volumeSum = 0
        for (const volume of volumes) volumeSum += volume
        const averageVolume = volumeSum / volumes.length
        // Value range: 127 = analyser.maxDecibels - analyser.minDecibels;
        this.analyserPercentage = Math.floor((averageVolume * 100) / 127)
      }, 100)
    },
    timerFn() {
      this.timerInterval = setInterval(() => {
        let difference = new Date().getTime() - this.startTime.getTime()

        let hours = Math.floor(
          (difference % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60),
        )
        let minutes = Math.floor((difference % (1000 * 60 * 60)) / (1000 * 60))
        let seconds = Math.floor((difference % (1000 * 60)) / 1000)
        let milliseconds = Math.floor((difference % (1000 * 60)) / 10)
        hours = hours < 10 ? '0' + hours : hours
        minutes = minutes < 10 ? '0' + minutes : minutes
        seconds = seconds < 10 ? '0' + seconds : seconds
        milliseconds =
          milliseconds < 100
            ? milliseconds < 10
              ? '00' + milliseconds
              : '0' + milliseconds
            : milliseconds
        this.timer = hours + ':' + minutes + ':' + seconds + ':' + milliseconds
      }, 10)
    },
  },
  components: {
    SvgIcon,
  },
}
</script>

<style lang="scss">
.container {
  font-family: 'Courier New', Courier, monospace;
  display: flex;
  flex-flow: row wrap;
}
.row {
  display: flex;
  justify-content: center;
  flex-basis: 100%;
  width: 100%;
}
.button {
  margin: 50px;
  padding: 25px;
  border-radius: 50%;
  background-color: whitesmoke;
  box-shadow: 1px 1px 5px 1px rgba(0, 0, 0, 0.3);
  cursor: pointer;
  transition: box-shadow 0.1s ease-in;
}

@for $i from 1 to 100 {
  .shadow-mic-#{$i} {
    -webkit-box-shadow: 0px 0px 1px #{$i / 2}px rgba(224, 127, 127, 0.3);
    -moz-box-shadow: 0px 0px 1px #{$i / 2}px rgba(224, 127, 127, 0.3);
    box-shadow: 0px 0px 1px #{$i / 2}px rgba(224, 127, 127, 0.3);
  }
}
</style>
