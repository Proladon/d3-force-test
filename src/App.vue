<template>
  <div id="app">
    <Home />
    <notifications position="top center" />
  </div>
</template>

<script>
import Home from '@/views/Home.vue'

export default {
  name: 'App',
  components: {
    Home,
  },
}
</script>

<style>
html,
body,
#app {
  @apply h-full w-full;
}
#app {
  font-family: 'Open Sans', Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
