<template>
  <div class="column">
    <VoiceRecorder />
  </div>
</template>

<script>
import VoiceRecorder from './components/VoiceRecorder.vue'

export default {
  name: 'App',
  components: {
    VoiceRecorder
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
  margin-top: 30px;
}
@media (orientation: landscape) {
  .column {
    display: inline-block;
    width: 100%;
  }
}
@media (orientation: portrait) {
  .column {
    display: inline-block;
    width: 100%;
  }
}
</style>
