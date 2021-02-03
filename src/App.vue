<template lang="pug">
  #app
    Recorder
</template>

<script>
import Recorder from './components/Recorder.vue'

export default {
  name: 'App',
  components: {
    Recorder,
  },
}
</script>

<style lang="scss">
body {
  margin: 0;
}
#app {
  height: 100vh;
  width: 100vw;
  background-color: white;
}
</style>
