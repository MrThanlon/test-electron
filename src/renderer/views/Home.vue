<template>
  <div>
    <svg class="graph" width="640" height="480" ref="svg" id="svg">
      <rect x="0" y="0" width="640" height="640" fill="#2a2a2e"></rect>
      <polyline points=""
                stroke-width="2"
                fill="none" stroke="#fca325"
                ref="line" id="line">
      </polyline>
    </svg>
    <div>
      <p>COM Port: {{portPath}}</p>
      <button @click="disconnectPort" :disabled="!connected">Disconnect</button>
      <button @click="pause">Pause</button>
      <span>{{drawingCounts}}</span>
    </div>
    <!--
    <canvas class="graph" id="canvas" width="1280" height="720" ref="canvas"></canvas>
    -->
  </div>
</template>

<script>
  import SerialPort from 'serialport'
  import { remote } from 'electron'

  export default {
    name: 'Home',
    data() {
      return {
        portPath: null,
        portScanInterval: 0,
        port: null,
        connected: false,
        queue: Buffer.from([]),
        running: false,
        points: null,
        drawingCounts: 0
      }
    },
    computed: {},
    mounted() {
      this.portScanInterval = setInterval(this.updatePortList, 1000)
      this.initPoints(1000)
      // this.test()
    },
    methods: {
      initPoints(length) {
        this.points = new Array(length)
        this.points.fill(0)
        const step = 640 / length
        this.points.forEach((v, idx, arr) => {
          arr[idx] = this.$refs.svg.createSVGPoint()
          arr[idx].x = step * idx
          arr[idx].y = -1
          this.$refs.line.points.appendItem(arr[idx])
        })
      },
      pause() {
        this.running = false
        console.debug(this.points)
      },
      continueRunning() {
        this.running = true
      },
      test() {
        this.points.forEach(v => { v.y = Math.random() * 10 + 230 })
      },
      async connectPort() {
        if (this.connected) {
          await this.disconnectPort()
        }
        this.running = true
        clearInterval(this.portScanInterval)
        await this.updatePortList()
        this.port = new SerialPort(this.portPath)
        this.port.on('data', e => {
          if (!this.running) {
            return
          }
          // e is Buffer
          this.queue = Buffer.concat([this.queue, e])
          // TODO: Trig Mode
          if (this.queue.length >= 1000) {
            this.port.flush()
            this.draw()
            this.running = false
            setTimeout(() => { this.running = true }, 50)
          }
        })
        this.connected = true
      },
      async disconnectPort() {
        if (!this.connected) {
          return
        }
        this.portScanInterval = setInterval(this.updatePortList, 1000)
        this.portPath = null
        this.port.close()
        await this.updatePortList()
        this.points.forEach(v => { v.y = -1 })
        this.connected = false
      },
      draw() {
        this.drawingCounts += 1
        this.points.forEach((v, idx) => {
          v.y = 400 - this.queue[idx]
        })
        this.queue = Buffer.from([])
      },
      async updatePortList() {
        const isMac = process.platform === 'darwin'
        const list = await SerialPort.list()
        console.debug(list)
        const menu = [
          ...(isMac ? [{
            label: 'Electron',
            submenu: [
              { role: 'about' },
              { type: 'separator' },
              {
                role: 'services',
                submenu: []
              },
              { type: 'separator' },
              { role: 'hide' },
              { role: 'hideothers' },
              { role: 'unhide' },
              { type: 'separator' },
              { role: 'quit' }
            ]
          }] : []),
          {
            label: 'Ports',
            submenu: list.length ? list.map(v => ({
              label: v.path,
              type: 'checkbox',
              checked: v.path === this.portPath,
              click: item => {
                console.debug(item.label)
                this.portPath = item.label
                this.connectPort()
              }
            })) : [
              { label: 'There is no Serial Port to be selected.', enabled: false }
            ]
          },
          {
            label: 'Edit',
            submenu: [
              { role: 'undo' },
              { role: 'redo' },
              { type: 'separator' },
              { role: 'cut' },
              { role: 'copy' },
              { role: 'paste' },
              ...(isMac ? [
                { role: 'pasteAndMatchStyle' },
                { role: 'delete' },
                { role: 'selectAll' },
                { type: 'separator' },
                {
                  label: 'Speech',
                  submenu: [
                    { role: 'startspeaking' },
                    { role: 'stopspeaking' }
                  ]
                }
              ] : [
                { role: 'delete' },
                { type: 'separator' },
                { role: 'selectAll' }
              ])
            ]
          },
          {
            role: 'help',
            submenu: [
              {
                label: 'Learn More',
                click: async () => {
                  // eslint-disable-next-line import/no-extraneous-dependencies
                  const { shell } = require('electron')
                  await shell.openExternal('https://blog.ch34k.xyz')
                }
              }
            ]
          }
        ]
        remote.Menu.setApplicationMenu(remote.Menu.buildFromTemplate(menu))
      }
    },
  }
</script>

<style scoped>
  .graph {
  }
</style>
