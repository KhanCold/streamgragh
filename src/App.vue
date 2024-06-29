<template>
  <div id="app">
    <!-- <img alt="Vue logo" src="./assets/logo.png"> -->
    <StreamGragh></StreamGragh>
  </div>
</template>

<script>

import StreamGragh from './components/StreamGragh.vue';
export default {
  name: 'App',
  components: {
   StreamGragh,
  }
}
</script>

<style>

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

#app {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

</style>
