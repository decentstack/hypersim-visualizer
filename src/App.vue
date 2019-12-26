<template>
  <div id="app">
    <samp id="tag"><a href="https://github.com/decentstack/hypersim-visualizer">@decentstack/hypersim-visualizer</a></samp>
    <NetGraph/>
  </div>
</template>

<script>
import NetGraph from './components/NetGraph.vue'

export default {
  name: 'app',
  components: {
    NetGraph
  }
}
</script>

<style lang="scss">
body {
  // background-color: #32383E;
  // color: #eee;
  margin: 0;
  padding: 0;
}

#tag {
  padding-left: 4px;
  font-size: 15pt;
  a {
    text-decoration: none;
    color: #2c3e50;
  }
}

#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}
</style>
