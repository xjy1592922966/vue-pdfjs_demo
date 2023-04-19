<template>
  <div id="app">
    <pdfView></pdfView>
  </div>
</template>

<script>
import pdfView from './page/pdf/pdfView.vue'

export default {
  name: 'App',
  components: {
    pdfView
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
