<template>
  <div>
    <webrtc-media></webrtc-media>
    <!-- <webrtcMediaDelay></webrtcMediaDelay> -->
  </div>
</template>
<script>
import webrtcMedia from "./views/webrtcMedia.vue";
import webrtcMediaDelay from "./views/webrtcMedia_delay.vue";
export default {
  components: { webrtcMedia},
};
</script>
<style  lang="scss">
body {
  padding: 0;
  margin: 0;
}
</style>