<template>
  <v-row justify="center">
    <v-col cols="8" md="8" sm="12" xs="12">
      <v-card>
        <v-card-title class="headline">
          Detect sitting or standing every 5 seconds.
        </v-card-title>
        <v-card-actions>
          <v-btn
            v-if="!isCaptureCycleRunning"
            class="card-action-button"
            color="#15B8A4"
            @click="startCapture()"
          >
            Start detection
          </v-btn>
          <v-btn
            v-if="isCaptureCycleRunning"
            class="card-action-button"
            color="#15B8A4"
            @click="endCapture()"
          >
            Finish detection
          </v-btn>
        </v-card-actions>
        <v-card-text class="flex-center">
          <video
            id="video"
            ref="video"
            width="600"
            height="500"
            autoplay
          ></video>
          <canvas
            id="overcanvas"
            ref="overcanvas"
            width="600"
            height="500"
          ></canvas>
          <canvas id="canvas" ref="canvas" width="600" height="500"></canvas>
        </v-card-text>
      </v-card>
    </v-col>
    <v-col cols="4" md="4" sm="12" xs="12">
      <v-card>
        <v-card-title class="headline"> Detection history </v-card-title>
        <ol>
          <li v-for="(c, cKey) in reverseCaptures" :key="c.d" class="capture">
            <img :src="c" class="capture-img" />
            <div class="capture-text">
              <span class="capture-num">{{ captures.length - cKey }}.</span>
              <span class="capture-judge">
                {{
                  judgments[captures.length - cKey - 1]
                    ? judgments[captures.length - cKey - 1]
                    : ''
                }}
              </span>
            </div>
          </li>
        </ol>
      </v-card>
    </v-col>
  </v-row>
</template>

<script>
import axios from 'axios'
export default {
  data() {
    return {
      video: {},
      canvas: {},
      overcanvas: {},
      captures: [],
      isCaptureCycleRunning: false,
      intervalID: null,
      captureCycleTime: 5000,
      judgments: [],
    }
  },
  computed: {
    reverseCaptures() {
      const cap = this.captures
      return cap.reverse()
    },
  },
  mounted() {
    this.video = this.$refs.video
    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
      navigator.mediaDevices.getUserMedia({ video: true }).then((stream) => {
        this.video.srcObject = stream
        this.video.play()
      })
    }
  },
  beforeDestroy() {
    clearInterval(this.intervalID)
  },
  methods: {
    startCapture() {
      this.isCaptureCycleRunning = true
      this.capture()
      const time = this.captureCycleTime
      this.intervalID = setInterval(() => {
        this.capture()
      }, time)
    },
    endCapture() {
      this.isCaptureCycleRunning = false
      clearInterval(this.intervalID)
    },
    capture() {
      this.canvas = this.$refs.canvas
      this.canvas.getContext('2d').drawImage(this.video, 0, 0, 600, 500)
      this.captures.push(this.canvas.toDataURL('image/png'))
      console.log('Captured!')
      this.sendImage()
    },
    sendImage() {
      const subscriptionKey = process.env.azureFaceKey1
      const endpoint = process.env.azureFaceEndpoint
      const uriBase = `${endpoint}face/v1.0/detect`
      // 配列最後に追加された画像のフォーマットを変換し、imgURL変数に代入する
      const imgURL = this.makeblob(this.captures[this.captures.length - 1])
      // imgURLの画像をFaceAPIに送信
      axios
        .post(
          uriBase +
            '?returnFaceId=true&returnFaceLandmarks=false&recognitionModel=recognition_02&returnRecognitionModel=false&detectionModel=detection_02',
          imgURL,
          {
            headers: {
              'Content-Type': 'application/octet-stream',
              'Ocp-Apim-Subscription-Key': subscriptionKey,
            },
          }
        )
        .then((response) => {
          console.log(response.data[0])
          if (response.data[0]) {
            this.writeBoundingBox(response.data[0])
            this.standUpCheck(response.data[0])
          } else {
            console.log('Nobody is here')
            this.judgments.push('Nobody is here')
          }
        })
        .catch((error) => {
          console.error(error)
        })
    },
    makeblob(dataURL) {
      const BASE64_MARKER = ';base64,'
      if (!dataURL.includes(BASE64_MARKER)) {
        const parts = dataURL.split(',')
        const contentType = parts[0].split(':')[1]
        const raw = decodeURIComponent(parts[1])
        return new Blob([raw], { type: contentType })
      }
      const parts = dataURL.split(BASE64_MARKER)
      const contentType = parts[0].split(':')[1]
      const raw = window.atob(parts[1])
      const rawLength = raw.length
      const uInt8Array = new Uint8Array(rawLength)
      for (let i = 0; i < rawLength; ++i) {
        uInt8Array[i] = raw.charCodeAt(i)
      }
      return new Blob([uInt8Array], { type: contentType })
    },
    writeBoundingBox(data) {
      this.overcanvas = this.$refs.overcanvas
      const ctx = this.overcanvas.getContext('2d')
      ctx.clearRect(0, 0, 600, 500)
      ctx.beginPath()
      ctx.rect(
        data.faceRectangle.left,
        data.faceRectangle.top,
        data.faceRectangle.width,
        data.faceRectangle.height
      )
      ctx.strokeStyle = '#15B8A4'
      ctx.lineWidth = 1
      ctx.stroke()
    },
    standUpCheck(data) {
      if (data.faceRectangle.width < 100) {
        console.log('You are standing')
        this.judgments.push('Standing')
      } else {
        console.log('You are sitting')
        this.judgments.push('Sitting')
      }
    },
  },
}
</script>
<style>
#canvas {
  display: none;
}
.flex-center {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  width: 100%;
  height: 500px;
}
#video {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  margin: auto;
  display: block;
}
#overcanvas {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  margin: auto;
  display: block;
}
.card-action-button {
  margin: 0 8px;
}
.capture {
  /* display: inline; */
  display: flex;
  align-items: center;
  margin: 8px 0;
}
.capture-text {
  padding-left: 20px;
}
.capture-num {
  min-width: 56px;
  font-size: 20px;
}
.capture-img {
  height: 120px;
}
.capture-judge {
  font-size: 20px;
}
</style>
