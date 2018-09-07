<template>
  <div class="search-box">
    <img src="../assets/Magnify-2s-200px.gif" />
  </div>
</template>

<script>
export default {
  name: 'search'
}
</script>

<style scoped>
  .search-box {
    width: 100%;
    height: 100%;
  }
  img {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    top: 0;
    margin: auto;
    width: 200px;
    height: 200px;
  }
</style>
