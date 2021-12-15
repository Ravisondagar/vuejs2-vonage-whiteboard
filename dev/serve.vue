<script>
import Vue from 'vue';
import Vue2VonageWhiteboard from '@/vue2-vonage-whiteboard.vue';

export default Vue.extend({
  name: 'ServeDev',
  components: {
    Vue2VonageWhiteboard
  }
});
</script>

<template>
  <div id="app">
    <vue2-vonage-whiteboard width="1500" height="500" />
  </div>
</template>
