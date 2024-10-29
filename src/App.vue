<template>
  <div>
    <Editor :textColor="textColor" :baseImageOptions="{ src: '../public/testShirt-filled.png' }"/>
    <div class="text-color-editor">
      Цвет текста
      <input type="color" v-model="textColor">
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import Editor from './components/Editor.vue'

const textColor = ref('#000000');
</script>

<style lang="scss"></style>