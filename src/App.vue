<template>
  <img :src="smallImg" alt="小于10kb的图片" />
  <img :src="bigImg" alt="大于于10kb的图片" />
  <!-- 小图片背景容器 -->
  <div className='smallImg'></div>
  <!-- 大图片背景容器 -->
  <div className='bigImg'></div>
  <div> 修改App.vue</div>
  <!-- 使用Demo1组件 -->
  <Demo1 />
</template>

<script setup lang="ts">
  import smallImg from './assets/imgs/5kb.png'
  import bigImg from './assets/imgs/22kb.png'
  import './app.css'
  import './app.less'
  import { Demo1, Demo2 } from '@/components' //引入Demo1和Demo2组件
</script>

<style scoped>
</style>

