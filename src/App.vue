<template>
  <div id="app">
    <Test />
  </div>
</template>

<script>
import Test from './components/Test';

export default {
  name: 'App',
  components: {
    Test
  }
}
</script>

<style>
html, body, #app {
  height: 100%
}
</style>
