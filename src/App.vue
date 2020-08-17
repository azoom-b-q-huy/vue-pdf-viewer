<template>
  <PDFViewer :url="url" rotate openNewTab download />
</template>

<script>
import PDFViewer from './components/PDFViewer'

const DEFAULT_URL =
  'https://storage.googleapis.com/my-awesome-project-212504.appspot.com/issue-attachments/04dcd39e-1ef2-469a-8a54-e11779571c5d.pdf'

export default {
  name: 'App',
  components: {
    PDFViewer,
  },
  data() {
    return {
      url: DEFAULT_URL,
    }
  },
}
</script>

<style></style>
