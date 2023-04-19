<template>
  <div>
    <PdfPreview  :url="pdfUrl" 
      style='width: 100%; height: 100%;' :scale="1" ref="PdfPreview"></PdfPreview>
  </div>
</template>

<script>
  import PdfPreview from './components/PdfPreview'
  export default {
    components: { PdfPreview },
    data() {
      return {
        pdfUrl: 'http://oss-cdn.hcolor.pro/assets/pdf/1.pdf',
      }
    },
    async beforeCreate() {
    },
    created() {
    },
    methods: {
    },
  }
</script>