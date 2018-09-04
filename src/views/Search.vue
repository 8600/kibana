<template>
  <div class="search-bar">
    <input v-model="value"/>
    <div class="button"></div>
  </div>
</template>

<script>
  export default {
    name: 'search-bar',
    props: {
      value: String
    }
  }
</script>

<style scoped>
  .search-bar {
    width: 100%;
    height: 40px;
    display: flex;
  }
  .button {
    background-color: skyblue;
    width: 40px;
    height: 40px;
  }
  input {
    width: calc(100% - 40px);
  }
</style>
