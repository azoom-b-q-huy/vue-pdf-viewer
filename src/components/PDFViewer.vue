<template>
  <div class="pdf-container">
    <div v-show="!rendering" :class="['pdf-toolbar', `-right`]">
      <button
        v-if="rotate"
        class="action"
        :style="btnStyles"
        @click="rotateLeft"
      >
        <img src="@/assets/rotate_left.svg" alt="rotate_left" />
      </button>
      <button
        v-if="rotate"
        class="action"
        :style="btnStyles"
        @click="rotateRight"
      >
        <img src="@/assets/rotate_right.svg" alt="rotate_right" />
      </button>
      <button v-if="zoom" class="action" :style="btnStyles" @click="zoomIn">
        <img src="@/assets/zoom_in.svg" alt="zoom_in" />
      </button>
      <button v-if="zoom" class="action" :style="btnStyles" @click="zoomOut">
        <img src="@/assets/zoom_out.svg" alt="zoom_out" />
      </button>
      <button
        v-if="openNewTab"
        class="action"
        :style="btnStyles"
        @click="openFileInNewTab"
      >
        <img src="@/assets/open_in_new.svg" alt="open_in_new" />
      </button>
      <button
        v-if="download"
        class="action"
        :style="btnStyles"
        @click="downloadFile"
      >
        <img src="@/assets/cloud_download.svg" alt="cloud_download" />
      </button>
    </div>

    <div v-show="!rendering" id="pdf-view" class="pdf-view"></div>

    <div v-show="rendering" class="pdf-loader">
      <LoadingIndicator />
    </div>
  </div>
</template>

<script>
import pdfjsLib from 'pdfjs-dist'
pdfjsLib.GlobalWorkerOptions.workerSrc = require('pdfjs-dist/build/pdf.worker.entry')

import LoadingIndicator from './LoadingIndicator'

export default {
  name: 'PDFViewer',
  components: {
    LoadingIndicator,
  },
  props: {
    url: String,
    fileName: {
      type: String,
      default: 'United.pdf',
    },
    position: {
      type: String,
      default: 'right',
    },
    zoom: Boolean,
    rotate: Boolean,
    openNewTab: Boolean,
    download: Boolean,
  },
  data() {
    return {
      // Element
      pdfDoc: null,

      // Option
      scale: 2.14,
      rotateValue: 0,
      zoomPercent: 100,

      // State
      pageNumber: 1,
      rendering: false,
    }
  },
  computed: {
    isVertical() {
      return this.rotateValue === 0 || this.rotateValue % 180 === 0
    },
    btnStyles() {
      const styles = {}

      this.position === 'top'
        ? (styles.marginRight = '16px')
        : (styles.marginBottom = '16px')
      return styles
    },
  },
  watch: {
    url: function(url) {
      const canvasContainer = document.getElementById('pdf-view')
      canvasContainer.innerHTML = null
      pdfjsLib.getDocument(url).promise.then(
        (_pdfDoc) => {
          this.rendering = true
          this.pdfDoc = _pdfDoc
          this.pageNumber = 1

          this.renderPages()
        },
        function(error) {
          // PDF loading error
          console.error(error)
        }
      )
    },
  },
  mounted() {
    pdfjsLib.getDocument(this.url).promise.then(
      (_pdfDoc) => {
        this.rendering = true
        this.pdfDoc = _pdfDoc

        this.renderPages()
      },
      function(error) {
        // PDF loading error
        console.error(error)
      }
    )
  },
  methods: {
    renderPage() {
      const self = this
      const scale = this.scale
      this.pdfDoc.getPage(this.pageNumber).then(async function(page) {
        const viewport = page.getViewport({ scale })

        const canvasContainer = document.getElementById('pdf-view')
        const canvas = document.createElement('canvas')
        const ctx = canvas.getContext('2d')

        const renderContext = {
          viewport,
          canvasContext: ctx,
        }

        canvas.className = 'pdf-canvas'
        canvas.width = viewport.width
        canvas.height = viewport.height
        canvas.style.width = '40%'
        canvas.style.margin = '4px 0'

        await canvasContainer.appendChild(canvas)

        page.render(renderContext).promise.then(() => {
          self.pageNumber++
          if (self.pageNumber <= self.pdfDoc.numPages) {
            return self.renderPage()
          }
          return (self.rendering = false)
        })
      })
    },
    renderPages() {
      this.renderPage()
    },
    rotateLeft() {
      this.rotateValue -= 90
      this.transformPages()
    },
    rotateRight() {
      this.rotateValue += 90
      this.transformPages()
    },
    zoomIn() {
      if (this.zoomPercent === 150) return
      this.zoomPercent += 10
      this.transformPages()
    },
    zoomOut() {
      if (this.zoomPercent === 50) return
      this.zoomPercent -= 10
      this.transformPages()
    },
    transformPages() {
      const self = this
      document.getElementsByClassName('pdf-canvas').forEach((canvasElm) => {
        canvasElm.style.margin = self.isVertical ? '4px 0' : '0 4px'
        canvasElm.style.transform = `rotate(${this.rotateValue}deg)`

        canvasElm.style.width = `calc(40% * ${self.zoomPercent} / 100)`
      })
    },
    openFileInNewTab() {
      window.open(this.url)
    },
    downloadFile() {
      // const requestHeaders = new Headers()
      // requestHeaders.append('cache-control', 'no-cache')
      // requestHeaders.append('pragma', 'no-cache')

      // const request = new Request(this.url, {
      //   method: 'GET',
      //   headers: requestHeaders,
      //   mode: 'no-cors'
      // })

      fetch(this.url)
        .then((response) => response.blob())
        .then((blob) => {
          const fileUrl = window.URL.createObjectURL(new Blob([blob]))

          const fileLink = document.createElement('a')

          fileLink.href = fileUrl
          fileLink.setAttribute('download', this.fileName)
          document.body.appendChild(fileLink)

          fileLink.click()
        })
        .catch((error) => console.error(error))
    },
  },
}
</script>

<style scoped lang="scss">
.pdf-container {
  background-color: rgba(33, 33, 33, 0.26);
  width: 100%;
  height: 100%;
  position: relative;

  > .pdf-view {
    overflow: hidden;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .pdf-toolbar {
    z-index: 1;
    position: fixed;
    display: flex;
    justify-content: center;
    align-items: center;

    &.-right {
      justify-content: center;
      flex-direction: column;
      height: 100%;
      right: 16px;
      bottom: 0;
    }

    &.-top {
      height: 48px;
      top: 0;
      left: 0;
      width: 100%;
    }

    > .action {
      background-color: transparent;
      border-width: 0px;
      width: 36px;
      height: 36px;
      display: flex;
      outline: none;
      justify-content: center;
      align-items: center;
      padding: 4px;
      border-radius: 50%;
    }
  }

  > .pdf-loader {
    width: 100vw;
    height: 100vh;
    background-color: inherit;
    display: flex;
    align-items: center;
    justify-content: center;
  }
}
</style>
