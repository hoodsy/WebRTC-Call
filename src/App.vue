<template>
  <div id="app">
    <img alt="Vue logo" src="./assets/logo.png" />
    <button v-on:click="toggleChat">Switch Video Client</button>
    <h2>Current Client: {{ active === 0 ? '1-1' : '2+' }}</h2>
    <HelloWorld v-if="active === 0" />
    <MultiChat v-if="active === 1" />
  </div>
</template>

<script>
import HelloWorld from "./components/HelloWorld.vue";
import MultiChat from "./components/MultiChat.vue";

export default {
  name: "App",
  components: {
    HelloWorld,
    MultiChat
  },
  data() {
    return {
      active: 1
    };
  },

  methods: {
    toggleChat() {
      if (this.active === 0) {
        this.active = 1;
      } else {
        this.active = 0;
      }
    }
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
