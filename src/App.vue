<template>
  <div id="app">
    <div>
      <div v-if="isHttps">
        <video id="video" :width="vWidth" :height="vHeight" style="border: 1px solid gray"></video>
        <button @click="enableScan">扫码</button>
      </div>
      <div v-else>
        <input type="text" v-model="httpsHost" width="100" @change="saveHost('https')" />
        <a :href="httpsJumpLink" target="_blank" rel="noopener noreferrer">跳转扫码</a>
      </div>
    </div>
    <div>
      <textarea name="result" cols="50" rows="3" v-model="scanResult"></textarea>
      <a
        :href="httpJumpLink"
        v-if="httpJumpLink && isHttps"
        target="_blank"
        rel="noopener noreferrer"
      >跳转查询</a>
    </div>
    <div v-if="!isHttps">
      <div>Fetch header</div>
      <div>
        <button @click="query">查询</button>
      </div>
      <div>
        <textarea name="result" cols="30" rows="10" v-model="queryResult"></textarea>
      </div>
    </div>
  </div>
</template>

<script>
import { BrowserQRCodeReader } from '@zxing/library'
import { ref, computed } from 'vue'

export default {
  components: {},
  name: 'App',
  setup() {
    const httpsHost = ref(window.localStorage.getItem('httpsHost') ?? '')
    const httpHost = ref(window.localStorage.getItem('httpHost') ?? '')
    const httpJumpLink = ref('')
    const httpsJumpLink = ref(
      httpsHost.value ? 'https://' + httpsHost.value : ''
    )

    const scanResult = ref('')
    let params = new URL(window.location).searchParams
    if (params.get('code')) {
      scanResult.value = decodeURIComponent(params.get('code'))
      httpJumpLink.value =
        'http://' +
        httpHost.value +
        '/?code=' +
        encodeURIComponent(scanResult.value.toString())
    }

    const qrscan = new BrowserQRCodeReader()

    const imageUrl = ref('')

    const imageAdded = (e) => {
      if (e.target.files.length == 0) {
        return
      }
      let file = e.target.files[0]
      let tempImage = new Image()

      tempImage.src = URL.createObjectURL(file)
      tempImage.onload = () => {
        console.log(tempImage.naturalWidth)
        console.log(tempImage.naturalHeight)
        let width = Math.min(tempImage.naturalWidth, 300)
        let height = (tempImage.naturalHeight / tempImage.naturalWidth) * width
        console.log(width)
        console.log(height)

        let canvas = document.createElement('canvas')
        canvas.width = width
        canvas.height = height

        canvas.getContext('2d').drawImage(tempImage, 0, 0, width, height)
        imageUrl.value = canvas.toDataURL('image/jpeg')
        URL.revokeObjectURL(tempImage.src)
      }
    }

    let vWidth = ref(300)
    let vHeight = ref(200)
    const enableScan = async () => {
      const media = await navigator.mediaDevices
        .getUserMedia({
          video: {
            facingMode: 'environment',
          },
        })
        .then((media) => {
          let { width, height } = media.getTracks()[0].getSettings()
          let newHeight = Math.min(height, 300)
          let newWidth = (width * newHeight) / height
          vWidth.value = newWidth
          vHeight.value = newHeight
          return media
        })
      qrscan.decodeOnceFromStream(media, 'video').then((value) => {
        // qrscan.decodeFromImageUrl(imageUrl.value).then((value) => {
        scanResult.value = value
        qrscan.stopStreams()
        httpJumpLink.value =
          'http://' + httpHost.value + '/?code=' + encodeURIComponent(value)
      })
    }

    const queryResult = ref('')

    const query = async () => {
      let string = scanResult.value.toString()
      if (string && string.lastIndexOf('#') > 0) {
        let code = string.substring(string.lastIndexOf('#') + 1)
        let res = await fetch(
          'http://wxcx.mem.gov.cn/certsearch/cert/scanQrcode',
          {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
            },
            body: JSON.stringify({ qrcode: code }),
          }
        )

        queryResult.value = (await res.json()).content.certNum
      }
    }

    const isHttps = computed(() => {
      return window.location.protocol == 'https:'
    })

    return {
      isHttps,
      vWidth,
      vHeight,
      imageUrl,
      imageAdded,
      qrscan,
      enableScan,
      scanResult,
      httpsHost,
      httpHost,
      httpJumpLink,
      httpsJumpLink,
      // get info
      queryResult,
      query,
    }
  },

  methods: {
    saveHost(option) {
      if (option == 'https') {
        window.localStorage.setItem('httpsHost', this.httpsHost)
      } else {
        window.localStorage.setItem('httpHost', this.httpHost)
      }
      this.httpsJumpLink = 'https://' + this.httpsHost
      this.httpJumpLink =
        'http://' +
        this.httpHost +
        '/?code=' +
        encodeURIComponent(this.scanResult)
    },
  },
}
</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
