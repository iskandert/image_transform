<template>
  <div class="container">
    <canvas ref="canvas"></canvas>

    <div class="form">
      <div class="left">
        <h4>Выберите исходную картинку</h4>
        <div class="form-item">
          <label for="file">Загрузить с устройства:</label>
          <input type="file" id="file" @change="previewImage" />
        </div>
        <div class="divider">
          <span>или</span>
        </div>
        <div class="form-item">
          <label for="link">Открыть по ссылке:</label>
          <input type="text" id="link" v-model="imageLink" />
        </div>
        <div class="divider">
          <span>или</span>
        </div>
        <div class="form-item">
          <label for="buffer">Вставить из буфера обмена:</label>
          <input type="text" id="buffer" @paste="pasteImage" />
        </div>
      </div>
      <div class="right">
        <h4>Задайте необходимую детализацию</h4>
        <div class="form-item">
          <label for="fragmentSize">Размер ячейки</label>
          <div class="input_container">
            <input type="range" min="1" max="100" v-model="fragmentSize" />
            <input type="number" id="fragmentSize" min="1" max="100" v-model="fragmentSize" />
            <span>px</span>
          </div>
        </div>
        <div class="form-item">
          <label for="pathSize">Размер символа относительно ячейки</label>
          <div class="input_container">
            <input type="range" min="1" max="100" v-model="pathSize" />
            <input type="number" id="pathSize" min="1" max="100" v-model="pathSize" />
            <span>%</span>
          </div>
        </div>
        <div class="form-item">
          <label for="pathWidth">Толщина обводки</label>
          <div class="input_container">
            <input type="range" min="1" max="100" v-model="pathWidth" />
            <input type="number" id="pathWidth" min="1" max="100" v-model="pathWidth" />
            <span>px</span>
          </div>
        </div>
        <div class="form-item">
          <label for="bright">Граница между светом и полутенью</label>
          <div class="input_container">
            <input type="range" :min="minBright" max="255" v-model="bright" />
            <input type="number" id="bright" :min="minBright" max="255" v-model="bright" />
          </div>
        </div>
        <div class="form-item">
          <label for="shadow">Граница между полутенью и тенью</label>
          <div class="input_container">
            <input type="range" min="1" :max="maxShadow" v-model="shadow" />
            <input type="number" id="shadow" min="1" :max="maxShadow" v-model="shadow" />
          </div>
        </div>
        <button class="save_button" @click="saveSVG">
          Скачать SVG
        </button>
      </div>
    </div>
    <div class="images">

      <div class="after">
        <h4>Результат</h4>
        <div class="img_container">
          <svg ref="svg" xmlns="http://www.w3.org/2000/svg" class="svg" :viewBox="viewBox" :width="imageWidth"
            :height="imageHeight">
            <path v-for="(path, idx) in pathItems" :key="idx" :d="path.d" :stroke-width="pathWidth" :stroke="path.stroke">
            </path>
          </svg>
        </div>
      </div>
      <div class="before">
        <h4>Исходная картинка</h4>
        <div class="img_container">
          <img v-if="imageUrl || imageLink" :src="imageUrl || imageLink" ref="before_image" alt="Preview Image" />
          <div v-else class="alt"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      file: null,
      imageUrl: null,
      imageLink: null,
      fragmentSize: 16,
      pathSize: 80,
      pathWidth: 1,
      bright: 100,
      shadow: 140,
      imageWidth: null,
      imageHeight: null,
      symbols: [],
      context: null,
      canCalculate: false
    }
  },
  computed: {
    viewBox() {
      return `0 0 ${this.imageWidth || 300} ${this.imageHeight || 100}`
    },
    pathItems() {
      return this.symbols
    },
    settings() {
      return {
        fragmentSize: this.fragmentSize,
        pathSize: this.pathSize,
        bright: this.bright,
        shadow: this.shadow,
      }
    },
    minBright() {
      return Math.max(+this.shadow + 1, 2)
    },
    maxShadow() {
      return Math.min(+this.bright - 1, 254)
    },
  },
  methods: {
    previewImage(event) {
      const input = event.target
      if (input.files && input.files[0]) {
        return this.file = input.files[0]
      }
      this.file = null
    },
    getPath({ x, y }, name) {
      const step = this.fragmentSize
      const size = step / 100 * this.pathSize
      const gap = (step - size) / 2

      if (name === 'plus') return {
        d: `M${x + step / 2},${y + gap} v${size} m-${size / 2},-${size / 2} h${size}`,
        name: 'plus',
        stroke: '#00ff15'
      }
      if (name === 'minus') return {
        d: `M${x + gap},${y + step / 2} h${size}`,
        name: 'minus',
        stroke: '#ff0000'
      }
    },
    pasteImage(e) {
      const items = Array.from((e.clipboardData || e.originalEvent.clipboardData).items)
      const item = items.find(item => item.type.indexOf('image') !== -1)
      this.file = item.getAsFile()
    },
    drawExternalImage() {
      let xhr = new XMLHttpRequest()
      xhr.open('GET', this.imageLink, true)
      xhr.responseType = 'blob'

      xhr.onload = (e) => {
        if (e.target.status === 200) {
          let blobImg = new Blob([e.target.response], { type: 'image/jpeg' })
          this.drawImage(blobImg)
        }
      };
      xhr.onerror = async (e) => {
        console.log(e);
        alert('Сервер не отвечает. Попробуйте скопировать картинку и вставить в другое поле из буфера обмена')
      }
      xhr.send();
    },
    drawImage(file) {
      const img = new Image()
      img.onload = () => {
        this.$refs.canvas.width = this.imageWidth = img.width
        this.$refs.canvas.height = this.imageHeight = img.height
        this.context.drawImage(img, 0, 0)
      };
      img.src = URL.createObjectURL(file)
      this.setSvg()
    },
    saveSVG() {
      const svg = this.$refs.svg
      const svgBlob = new Blob([svg.outerHTML], { type: 'image/svg+xml' });
      const svgUrl = URL.createObjectURL(svgBlob)
      const downloadLink = document.createElement('a')

      downloadLink.href = svgUrl
      downloadLink.download = 'my-svg-file.svg'
      document.body.appendChild(downloadLink)
      downloadLink.click()
      document.body.removeChild(downloadLink)
    },
    async setSvg() {
      this.canCalculate = false
      let a = new Date()
      let symbols = []
      const svg = this.$refs.svg
      const canvas = this.$refs.canvas
      const size = this.fragmentSize

      const rows = Math.ceil(canvas.height / size)
      const cols = Math.ceil(canvas.width / size)

      svg.setAttribute('width', canvas.width)
      svg.setAttribute('height', canvas.height)
      for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
          const x = j * size
          const y = i * size
          const imageData = this.context.getImageData(x, y, size, size)

          let blackPixels = 0
          let greyPixels = 0
          let whitePixels = 0
          for (let k = 0; k < imageData.data.length; k += 4) {
            const r = imageData.data[k]
            const g = imageData.data[k + 1]
            const b = imageData.data[k + 2]
            const brightness = (r * 0.2126) + (g * 0.7152) + (b * 0.0722)
            if (brightness <= this.shadow) {
              blackPixels++
            } else if (brightness > this.shadow && brightness <= this.bright) {
              greyPixels++
            } else {
              whitePixels++
            }
          }
          if (blackPixels > whitePixels && blackPixels > greyPixels) {
            symbols.push({})
          } else if (greyPixels > whitePixels) {
            symbols.push(this.getPath({ x, y }, 'minus'))
          } else {
            symbols.push(this.getPath({ x, y }, 'plus'))
          }
        }
      }
      this.symbols = symbols
      this.canCalculate = true
      console.log(new Date() - a);
    },
  },
  watch: {
    file: {
      handler(nv) {
        if (nv) {
          this.imageLink = null
          this.symbols = []
          const reader = new FileReader()
          reader.onload = e => {
            this.imageUrl = e.target.result
          }
          reader.readAsDataURL(nv)
          return this.drawImage(nv)
        }
        this.imageUrl = null
        this.symbols = []
      },
      deep: true
    },
    imageLink: {
      handler(nv) {
        if (nv) {
          this.imageUrl = null
          this.symbols = []
          this.drawExternalImage()
        }
      },
      deep: true
    },
    settings: {
      async handler(nv) {
        if (this.canCalculate) this.setSvg()
      },
      deep: true
    },
    async canCalculate(nv) {
      if (nv) this.setSvg()
    }
  },
  mounted() {
    this.context = this.$refs.canvas.getContext('2d', { willReadFrequently: true })
  },
}

</script>

<style scoped>
canvas {
  position: absolute;
  left: 2000vw;
}

.container {
  display: flex;
  flex-wrap: wrap;
  gap: 24px;
}

.images {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.form {
  display: flex;
  gap: 24px;
  flex-direction: column;
}

.form-item {
  display: flex;
  flex-direction: column;
  align-items: start;
  font-size: 14px;
  gap: 8px;
}

.input_container {
  display: flex;
  gap: 4px;
}

.input_container>input:first-of-type {
  margin-right: 12px;
}

.divider>span {
  display: inline-block;
  margin: 12px -5px;
}

.divider>span::after,
.divider>span::before {
  content: ' ';
  display: inline-block;
  width: 92px;
  height: .5px;
  margin: 0 5px 3px;
  background-color: black;
}

.before .img_container {
  width: 300px
}

.after .img_container {
  width: max-content;
  max-width: 50vw;
  max-height: 90vh;
}

.svg {
  background-color: black;
  width: 100%;
  height: min-content;
  max-height: 90vh;
}

@media (max-width: 992px) {
  .after .img_container {
    width: 100%;
  }

  .svg {
    width: 300px;
    height: auto;
    max-height: 90vh;
  }
}

.img_container>img {
  width: 100%;
  object-fit: contain;
}

.before>h4,
.after>h4 {
  margin-bottom: 8px;
}

.alt {
  height: 300px;
  background-color: rgb(214, 214, 214);
}


.save_button {
  margin-top: 8px;
  color: #fff;
  padding: 8px 15px;
  background-color: rgb(69, 142, 243);
  border: none;
  border-radius: 40px;
  cursor: pointer;
}

.save_button:hover {
  background-color: rgba(69, 142, 243, 0.774);
}
</style>
