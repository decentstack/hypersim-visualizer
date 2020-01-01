<template>
  <section>
    <span>
      <span style="float: right">Events loaded {{progress}}</span>
      Load swarm-log
      <input type="file" @change="onFileChange"/>
      {{current + 1}} / {{timeline.length}}
      <button @click="bkd">&lt;</button>
      <button @click="playPause">
        <span v-if="!isPlaying">-&gt;</span>
        <span v-if="isPlaying">||</span>
      </button>
      <button @click="fwd">&gt;</button>
    </span>
    <div style="background-color: #aaa">
      <samp>
        Peers: {{snap.stats.peers}},
        Pen: {{snap.stats.pending}},
        Con: {{snap.stats.connections}}
        BW: {{snap.stats.rate>>8}}KBps of {{snap.stats.capacity >> 8}}KBps
        t: {{Math.floor(snap.stats.time/1000)}}s
        ic: {{ (snap.stats.interconnection||0).toFixed(2) }}
      </samp>
    </div>
    <d3-network :net-nodes="snap.peers"
      :net-links="snap.links"
      :options="options"
      :node-cb="nodeFmt"
      :link-cb="linkFmt"/>
  </section>
</template>
<script>
// SPDX-License-Identifier: AGPL-3.0-or-later
import { defer } from 'deferinfer'
import { BasicTimeline } from 'hypersim-parser'
import D3Network from 'vue-d3-network'
import 'vue-d3-network/dist/vue-d3-network.css'
// import SOS from 'save-our-sanity'

function snapInit () {
  return {
    iteration: 0,
    stats: {
      pending: 0,
      connections: 0,
      interconnection: 0,
      peers: 0,
      capacity: 0,
      rate: 0,
      load: 0,
      time: 0,
      sessionId: 0
    },
    peers: [],
    links: [],
    custom: {}
  }
}

export default {
  components: { D3Network },
  data () {
    return {
      timeline: [],
      snap: {
        ...snapInit(),
        peers: [
          { id: 1, name: '1.red', _size: 10 },
          { id: 2, name: '2.blue', _size: 10 }
        ],
        links: [
          { sid: 1, tid: 2, name: 'label', load: 0.5 }
        ]
      },
      current: 0,
      options: {
        render: false,
        nodeSize: 33,
        linkWidth: 5,
        nodeLabels: true,
        linkLabels: true,
        forces: {
          X: 0.3,
          Y: 0.3
        }
      },
      isPlaying: 0,
      progress: 0
    }
  },
  methods: {
    fwd () { return this.gotoIdx((this.current + 1) % this.timeline.length) },
    bkd () { return this.gotoIdx(Math.max(0, this.current - 1)) },
    playPause () {
      if (!this.isPlaying) {
        const tick = () => {
          setTimeout(() => {
            if (this.isPlaying) {
              if (this.current === this.timeline.length - 1) {
                this.isPlaying = false
                return
              }
              this.fwd()
              tick()
            }
          }, 100)
        }
        tick()
      }
      this.isPlaying = !this.isPlaying
    },
    gotoIdx (n, forceReplace = false) {
      if (!this.isPlaying) console.log('Change to#', n)
      const p = this.current
      if (n === p && !forceReplace) return
      if (forceReplace || n < p) {
        this.$set(this, 'snap', this.timeline[n])
      } else {
        for (let i = p + 1; i <= n; i++) {
          if (!this.isPlaying) console.log('Merging snapshot#', i)
          const { stats, iteration, peers, links } = this.timeline[n]
          this.snap.iteration = iteration
          Object.assign(this.snap.stats, stats)
          for (const node of peers) {
            const pnode = this.snap.peers.find(nd => nd.id === node.id)
            if (pnode) Object.assign(pnode, node)
            else this.snap.peers.push(node)
          }
          for (const link of links) {
            const plink = this.snap.links.find(nd => nd.id === link.id)
            if (plink) Object.assign(plink, link)
            else this.snap.links.push(link)
          }
        }
      }
      this.current = n
    },
    linkFmt (link) {
      /* if (!link.load) {
        link._color = '#aaa'
      } else {
        const r = (link.load * 0xff).toString().padStart(2, '0')
        const g = (0xff - link.load * 0xff).toString().padStart(2, '0')
        debugger
        link._color = `#${r}${g}00`
      } */
      return link
    },
    nodeFmt (node) {
      node._size = 24
      return node
    },
    async onFileChange (ev) {
      var files = ev.target.files || ev.dataTransfer.files
      if (!files.length) return
      const timeline = []
      this.progress = 0
      const parser = new BasicTimeline()
      parser.pushReducer('peers', (val, ev, lut) => {
        if (ev.type === 'peer' && ev.event === 'tick') {
          const peer = lut[ev.id]
          peer.role = ev.name
          peer.name = `${peer.id}#${peer.role} @${peer.downloaded || -1}`
        }
        return val
      })

      // Attach sid/tid for d3-vue to pick them up.
      parser.pushReducer('links', (val, ev, lut) => {
        if (ev.type !== 'socket' || ev.event !== 'tick') return val
        const link = lut[ev.id]
        link.sid = ev.src
        link.tid = ev.dst
        link.name = `${link.rx} / ${link.tx}`
        return val
      })

      parser.on('snapshot', snapshot => timeline.push(snapshot))
      const f = files.item(0)
      const reader = f.stream().getReader()

      console.info('Begin parsing')
      while (true) {
        const { done, value } = await reader.read()
        // There's no time for ui updates inside the while loop
        // also chunks written != events parsed.
        // this.$set(this, 'progress', this.progress + 1)
        if (done) break
        await defer(d => parser._write(value, d))
      }
      this.$set(this, 'timeline', timeline)
      this.gotoIdx(0)
      // this.$set(this, 'progress', 0)
      console.info('Finished reading log')
    }
  }
}
</script>
<style>
/* @import 'vue-d3-network-plus/dist/vue-d3-network.css';*/
</style>
