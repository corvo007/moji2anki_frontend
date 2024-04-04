<template>
  <div id="app">
    <DeckBuilder />
  </div>
</template>

<script>
import DeckBuilder from './components/DeckBuilder.vue';

export default {
  name: 'App',
  components: {
    DeckBuilder
  }
}
</script>

<style>
#app {
  text-align: center;
  margin-top: 60px;
}
</style>
