<template>
  <section>
    <span>
      Load swarm-log
      <input type="file" @change="onFileChange"/>
      <button @click="bkd">prev</button>
      {{current + 1}} / {{timeline.length}}
      <button @click="fwd">next</button>
    </span>
    <div style="background-color: #aaa">
      <samp>
        Peers: {{snapshot.peers}},
        Pen: {{snapshot.pending}},
        Con: {{snapshot.connections}}
        BW: {{snapshot.rate>>8}}KBps of {{snapshot.capacity >> 8}}KBps
        t: {{Math.floor(snapshot.time/1000)}}s
      </samp>
    </div>
    <d3-network :net-nodes="snapshot.nodes"
      :net-links="snapshot.links"
      :options="options"
      :node-cb="nodeFmt"/>
  </section>
</template>
<script>
import { BasicTimeline } from 'hypersim-parser'
import D3Network from 'vue-d3-network'
import 'vue-d3-network/dist/vue-d3-network.css'
const CONNECTING = 0
const ACTIVE = 1

function snapInit () {
  return {
    iteration: 0,
    pending: 0,
    connections: 0,
    peers: 0,
    capacity: 0,
    rate: 0,
    load: 0,
    time: 0,
    sessionId: 0,
    nodes: [],
    links: [],
    custom: {}
  }
}

export default {
  components: { D3Network },
  data () {
    return {
      timeline: [{
        ...snapInit(),
        nodes: [
          { id: 1, name: '1.red', _size: 10 },
          { id: 2, name: '2.blue', _size: 10 }
        ],
        links: [
          { sid: 1, tid: 2, name: 'label' }
        ]
      }],
      current: 0,
      customNames: [],
      options: {
        render: false,
        nodeSize: 33,
        linkWidth: 10,
        nodeLabels: true,
        linkLabels: true,
        forces: {
          X: 0.3,
          Y: 0.3
        }
      }
    }
  },
  computed: {
    snapshot () { return this.timeline[this.current] || {} }
  },
  methods: {
    fwd () { this.current = (this.current + 1) % this.timeline.length },
    bkd () { this.current = Math.max(0, this.current - 1) },
    nodeFmt (node) {
      node._size = 24
      return node
    },
    onFileChange (ev) {
      var files = ev.target.files || ev.dataTransfer.files
      if (!files.length) return
      const f = files[0]
      f.text()
        .then(txt => txt.split('\n'))
        .then(lines => {
          const customNames = {}
          const timeline = []
          const _nodeCache = {}
          const _edgeCache = {}
          let iter = 0
          for (const line of lines) {
            if (!line.length) continue
            try {
              const l = JSON.parse(line)
              const evnt = l.event

              // Lazy set iteration
              if (typeof l.iteration !== 'undefined') {
                iter = l.iteration
              }

              // Lazy init new snapshot
              if (!timeline[iter]) {
                const prev = timeline[iter - 1]
                timeline[iter] = {
                  ...snapInit(),
                  nodes: prev ? [...timeline[iter - 1].nodes] : [],
                  links: []
                }
              }

              // Select correct snapshot for line
              const snap = timeline[iter]

              // Parse event.
              switch (l.type) {
                case 'peer':
                  switch (evnt) {
                    case 'init':
                      const n = {
                        id: l.id, name: `${l.id}.${l.name}`
                      }
                      _nodeCache[l.id] = n
                      snap.nodes.push(n)
                      break
                    case 'end':
                      if (_nodeCache[l.id]) _nodeCache[l.id].ended = true
                      break
                    default: console.warn('Unsupported event', l)
                  }
                  break
                case 'custom':
                  customNames[evnt] = true
                  if (!snap.custom[evnt]) snap.custom[evnt] = []
                  snap.custom[evnt].push(l)
                  break
                case 'socket':
                  switch (evnt) {
                    case 'tick':
                      snap.links.push({
                        sid: l.src,
                        tid: l.dst,
                        state: ACTIVE,
                        name: `rx/tx: ${l.rx}/${l.tx}`
                      })
                      break
                    case 'open':
                      const e = {
                        sid: l.src,
                        tid: l.dst,
                        state: CONNECTING,
                        name: `connecting...`
                      }
                      _edgeCache[e] = e
                      snap.links.push(e)
                      break
                    default: console.warn('Unsupported event', l)
                  }
                  break
                case 'simulator':
                  switch (evnt) {
                    case 'state-init':
                      // l.swarm // 'hypersim'
                    case 'create-cache':
                      break
                    case 'state-running':
                      // const { interval, speed } = l
                      break
                    case 'tick':
                      snap.time = l.time
                      snap.delta = l.delta
                      snap.pending = l.pending
                      snap.connections = l.connections
                      snap.rate = l.rate
                      break
                    default: console.warn('Unsupported event', l)
                  }
                  break
                default: console.warn('Unsupported event', l)
              }
            } catch (err) {
              console.warn('Failed parsing line: ', `'${line}'`, err)
            }
          }
          this.$set(this, 'customNames', Object.keys(customNames))
          this.$set(this, 'timeline', timeline)
          this.$set(this, 'current', 0)
        })
    }
  }
}
</script>
<style>
/* @import 'vue-d3-network-plus/dist/vue-d3-network.css';*/
</style>
