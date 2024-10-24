<template>
  <div>
    <Editor :textColor="textColor" />
    <div class="">
      Цвет текста
      <input type="color" v-model="textColor" @сhange="updateTextColor($event.target.value)">
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import Editor from './components/Editor.vue'

const textColor = ref('');
const updateTextColor = (newColor: string) => {
  textColor.value = newColor;
};
</script>

<style lang="scss"></style>