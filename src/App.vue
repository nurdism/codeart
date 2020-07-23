<template>
  <main class="grid grid-cols-2 h-screen">
    <div class="col-span-1 h-screen overflow-y-auto">
      <textarea class="h-screen w-full p-3 resize-none" placeholder="pase your src here..." v-model="src" />
    </div>
    <div class="col-span-1 flex flex-col h-screen">
      <div class="h-1/2 overflow-y-auto">
        <canvas ref="canvas" class="w-full h-auto" />
      </div>
      <pre
        ref="pre"
        class="flex-grow h-1/2 language-typescript overflow-y-auto"
        style="margin: 0;"
      ><code ref="code" class="language-typescript" v-html="highlight" /></pre>
    </div>
  </main>
</template>

<style scoped>
  canvas {
    image-rendering: optimizeSpeed;
    image-rendering: -moz-crisp-edges;
    image-rendering: -webkit-optimize-contrast;
    image-rendering: -o-crisp-edges;
    image-rendering: pixelated;
    -ms-interpolation-mode: nearest-neighbor;
  }

  ::selection {
    background: #73b4ff;
  }
</style>

<script lang="ts">
  import Prism from 'prismjs'

  import { Component, Vue, Watch, Ref } from 'vue-property-decorator'

  type Color = [number | null, number | null, number | null]
  type Line = Array<Color>

  const rgb = /rgb\((\d{1,3}), (\d{1,3}), (\d{1,3})\)/
  const whitespace = /\s/
  const newline = /\n/

  @Component({
    name: 'app',
  })
  export default class App extends Vue {
    @Ref('pre')
    pre!: HTMLPreElement

    @Ref('code')
    code!: HTMLElement

    @Ref('canvas')
    canvas!: HTMLCanvasElement

    src = ''
    highlight = ''

    @Watch('src')
    onSrcChanged(val: string) {
      try {
        this.highlight = Prism.highlight(val, Prism.languages.typescript, 'typescript')
      } catch (err) {
        console.log(err)
      }

      this.$nextTick(() => {
        try {
          this.image(this.src)
        } catch (err) {
          console.log(err)
        }
      })
    }

    image(src: string) {
      const lines: Array<Color | null> = []

      for (const token of Prism.tokenize(src, Prism.languages.typescript)) {
        this.process(token, lines, this.rgb(getComputedStyle(this.pre).color))
      }

      const image: Array<Line> = []

      let line: Line = []
      let max = 0
      let counter = 0

      for (const pixel of lines) {
        if (pixel === null) {
          image.push(line)
          line = []
          counter = 0
        } else {
          counter++
          if (max <= counter) {
            max = counter
          }
          line.push(pixel)
        }
      }

      image.push(line)

      this.draw(image, max, image.length, this.rgb(getComputedStyle(this.pre).backgroundColor))
    }

    draw(pixles: Array<Line>, width: number, height: number, background: Color = [0, 0, 0]) {
      this.canvas.width = width
      this.canvas.height = height

      const context = this.canvas.getContext('2d')
      if (context) {
        const data = context.createImageData(width, height)

        for (let x = 0; x < width; x++) {
          for (let y = 0; y < height; y++) {
            const i = (y * width + x) * 4
            if (
              pixles[y] &&
              pixles[y][x] &&
              pixles[y][x][0] !== null &&
              pixles[y][x][1] !== null &&
              pixles[y][x][2] !== null
            ) {
              const [r, g, b] = pixles[y][x]
              data.data[i] = r || 0
              data.data[i + 1] = g || 0
              data.data[i + 2] = b || 0
              data.data[i + 3] = 255
            } else {
              const [r, g, b] = background
              data.data[i] = r || 0
              data.data[i + 1] = g || 0
              data.data[i + 2] = b || 0
              data.data[i + 3] = 255
            }
          }
        }
        context.putImageData(data, 0, 0)
      }
    }

    process(token: string | Prism.Token, lines: Array<Color | null>, color: Color) {
      let text = ''

      if (typeof token !== 'string') {
        const element = document.querySelector(`.${token.type}`)
        if (element) {
          color = this.rgb(getComputedStyle(element).color)
        }

        if (Array.isArray(token.content)) {
          for (const subtoken of token.content) {
            this.process(subtoken, lines, color)
          }
        } else if (typeof token.content !== 'string') {
          this.process(token.content, lines, color)
        } else {
          text = token.content
        }
      } else {
        text = token
      }

      for (let i = 0; i < text.length; i++) {
        const char = text.charAt(i)
        if (newline.test(char)) {
          lines.push(null) // null === newline
        } else if (whitespace.test(char)) {
          lines.push([null, null, null]) // [null, null, null] whitespace
        } else {
          lines.push(color)
        }
      }
    }

    rgb(str: string): Color {
      const match = rgb.exec(str)
      if (match !== null) {
        return [parseInt(match[1]), parseInt(match[2]), parseInt(match[3])]
      }
      return [0, 0, 0]
    }
  }
</script>
