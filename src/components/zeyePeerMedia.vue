<template>
  <div
    class="zeye-peer-media"
    :class="{ 'active-speaker': $zeyeClient.isSpeakerActive(peerId) }"
  >
    <video
      ref="videoElem"
      autoPlay
      playsInline
      muted
      :controls="false"
      style="width: 100%"
    />
    <audio
      ref="audioElem"
      autoPlay
      playsInline
      muted
      :controls="false"
      style="width: 100%"
    />

    <div v-if="showVolumeBar" class="volume-container">
      <div
        class="bar"
        :style="{
          height: audioVolume * 10 + '%',
          backgroundColor: volumeBarColor
        }"
      />
    </div>
  </div>
</template>

<script>
import hark from 'hark'

export default {
  props: {
    peerId: {
      type: String,
      required: true
    },
    showVolumeBar: {
      type: Boolean,
      default: false
    },
    volumeBarColor: {
      type: String,
      default: 'greenyellow'
    }
  },
  data() {
    return {
      hark: null,
      audioVolume: 0 // Integer from 0 to 10
    }
  },
  computed: {
    isLocalMedia() {
      return this.peerId === 'me'
    }
  },
  mounted() {
    this.waitForMediaAvailability()

    if (this.isLocalMedia) {
      this.$zeyeClient.$bus.$on('update-my-media', () => {
        this.runVideo()
        this.runAudio()
      })
    }

    this.$zeyeClient.$bus.$on('set-output-device-id', (deviceId) => {
      this.$refs.audioElem.setSinkId(deviceId)
    })
  },
  beforeDestroy() {
    this.$zeyeClient.$bus.$off('update-my-media')
    this.$zeyeClient.$bus.$off('set-output-device-id')
  },
  methods: {
    runAudio() {
      // for nonMe peers there should be a Consumer getter
      let audioTrack

      if (this.isLocalMedia) {
        audioTrack = this.$zeyeClient.getAudioProducer().track
      } else {
        audioTrack = this.$zeyeClient.getAudioConsumer(this.peerId).track
      }

      if (audioTrack) {
        const { audioElem } = this.$refs

        audioElem.muted = this.isLocalMedia

        const audioStream = new MediaStream()

        audioStream.addTrack(audioTrack)
        audioElem.srcObject = audioStream

        audioElem
          .play()
          .catch((error) => console.warn('audioElem.play() failed:%o', error))

        this._runHark(audioStream)
      }
    },
    runVideo() {
      let videoTrack

      if (this.isLocalMedia) {
        videoTrack = this.$zeyeClient.getVideoProducer().track
      } else {
        videoTrack = this.$zeyeClient.getVideoConsumer(this.peerId).track
      }

      if (videoTrack) {
        const { audioElem, videoElem } = this.$refs

        videoElem.muted = true

        const videoStream = new MediaStream()

        videoStream.addTrack(videoTrack)
        videoElem.srcObject = videoStream

        videoElem.onplay = () => {
          audioElem
            .play()
            .catch((error) => console.warn('audioElem.play() failed:%o', error))
        }

        videoElem.play()
      }
    },
    _runHark(stream) {
      if (!stream.getAudioTracks()[0])
        throw new Error('_runHark() | given stream has no audio track')

      this.hark = hark(stream, { play: false })

      // eslint-disable-next-line no-unused-vars
      this.hark.on('volume_change', (dBs, threshold) => {
        // The exact formula to convert from dBs (-100..0) to linear (0..1) is:
        //   Math.pow(10, dBs / 20)
        // However it does not produce a visually useful output, so let exagerate
        // it a bit. Also, let convert it from 0..1 to 0..10 and avoid value 1 to
        // minimize component renderings.
        let audioVolume = Math.round(10 ** (dBs / 85) * 10)

        if (audioVolume === 1) audioVolume = 0

        if (audioVolume !== this.audioVolume) {
          this.audioVolume = audioVolume
        }
      })
    },

    waitForMediaAvailability() {
      if (
        (this.isLocalMedia &&
          this.$zeyeClient.getAudioProducer() &&
          this.$zeyeClient.getVideoProducer()) ||
        (!this.isLocalMedia &&
          this.$zeyeClient.getPeer(this.peerId) &&
          this.$zeyeClient.getPeer(this.peerId).consumers &&
          this.$zeyeClient.getAudioConsumer(this.peerId) &&
          this.$zeyeClient.getVideoConsumer(this.peerId))
      ) {
        this.runAudio()
        this.runVideo()
      } else {
        console.debug('Waiting for channels availability...')
        setTimeout(this.waitForMediaAvailability, 100)
      }
    }
  }
}
</script>

<style>
.volume-container {
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 10px;
  display: flex;
  -webkit-box-orient: vertical;
  flex-direction: column;
  -webkit-box-pack: center;
  justify-content: center;
  -webkit-box-align: center;
  align-items: center;
  pointer-events: none;
}

.volume-container .bar {
  width: 6px;
  border-radius: 6px;
  transition: 0.1s ease-in 0s;
}

.zeye-peer-media {
  position: relative;
  flex: 100 100 auto;
  display: flex;
}

.zeye-peer-media.active-speaker {
  box-shadow: 0 0 5px greenyellow;
}
</style>
