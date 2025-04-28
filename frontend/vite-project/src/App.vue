<template>
  <div class="appMain">
    <div class="tabPane">
      <span @click="curPage = 'quill'" :class="['tabItem', curPage === 'quill'? 'selected': '' ]">Quill</span>
      <span @click="curPage = 'prosemirror'" :class="['tabItem', curPage !== 'quill'? 'selected': '' ]">ProseMirror</span>
    </div>

    <QuillEditorPage v-if="curPage === 'quill'" />
    <ProseMirrorEditorPage v-else />
  </div>
</template>

<script setup lang="ts">
import QuillEditorPage from './components/QuillEditorPage.vue'
import ProseMirrorEditorPage from './components/ProseMirrorEditorPage.vue'
import { ref } from 'vue'

const curPage = ref<'quill' | 'prosemirror'>('quill')
</script>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}

.appMain {
  width: auto;
  height: auto;
  padding: 20px;
}

.tabPane {
  text-align: left;
}


.tabItem {
  cursor: pointer;
  margin-right: 10px;
}

.tabItem.selected {
  color: #05308d;
}

.tabItem:not(.selected):hover {
  color: #4174e0;
}
</style>
