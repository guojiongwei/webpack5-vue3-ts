<template>
  <img :src="smallImg" alt="小于10kb的图片" />
  <img :src="bigImg" alt="大于于10kb的图片" />
  <!-- 小图片背景容器 -->
  <div className='smallImg'></div>
  <!-- 大图片背景容器 -->
  <div className='bigImg'></div>
  <div> 修改App.vue</div>
</template>

<script setup lang="ts">
  import smallImg from './assets/imgs/5kb.png'
  import bigImg from './assets/imgs/22kb.png'
  import './app.css'
  import './app.less'
</script>

<style scoped>
</style>
