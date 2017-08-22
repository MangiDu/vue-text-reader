<template>
  <div id="app">
    <reader :text="text"></reader>
  </div>
</template>

<script>
import Reader from './components/reader/index'

export default {
  name: 'app',
  components: {
    Reader
  },
  data () {
    return {
      text: ''
    }
  },
  mounted () {
    let request = new XMLHttpRequest()
    request.onreadystatechange = () => {
      if (request.readyState === 4 && request.status === 200) {
        this.text = request.responseText
      }
    }
    request.open('GET', '/static/article.txt', true)
    request.send()
  }
}
</script>

<style>
html, body {
  margin: 0;
  padding: 0;
  height: 100%;
}
#app {
  height: 100%;
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}
</style>
