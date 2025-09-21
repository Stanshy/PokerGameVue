<template>
  <div class="home-view">
    <PokerGame />
  </div>
</template>

<script>
import PokerGame from '@/components/pokerG.vue'

export default {
  name: 'HomeView',
  components: {
    PokerGame
  }
}
</script>

<style scoped>
.home-view {
  width: 100%;
  min-height: 100vh;
  margin: 0;
  padding: 0;
}

/* 確保全螢幕顯示 */
* {
  box-sizing: border-box;
}
</style>