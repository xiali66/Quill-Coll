<template>
  <div class="userLine">
    <img class="avatarImg" :src="item.avatar" :title="item.name" v-for="(item, key) of onlineUserList" :key="key" />
  </div>
  <div ref="quillRef"></div>
</template>

<script setup lang="ts">
import Quill from 'quill'
import QuillCursors from 'quill-cursors'
import 'quill/dist/quill.snow.css'
import { onMounted, onUnmounted, ref } from 'vue'
import * as Y from 'yjs'
import { QuillBinding } from 'y-quill'
import { WebsocketProvider } from 'y-websocket'
// import QuillBetterTable from 'quill-better-table'
import 'quill-better-table/dist/quill-better-table.css'
import { tableEmbed } from './table-embed.js'
import TableEmbed from 'quill/modules/tableEmbed.js'

let quillObj = null as null | Quill
const quillRef = ref<HTMLElement>()
let provider = null as null | WebsocketProvider

Quill.register('modules/cursors', QuillCursors)
const ydoc = new Y.Doc()
// 在文档上定义共享文本类型
const ytext = ydoc.getText('quill')

const onlineUserList = ref<any[]>([])

TableEmbed.register()

const embeds = {
  'table-embed': tableEmbed
}

// 创建一个编辑器绑定 将quill编辑器“绑定”到 Y.Text 类型。
let binding = null
onMounted(() => {
  if (quillRef.value) {

    quillObj = new Quill(quillRef.value, {
      modules: {
        cursors: true,
        toolbar: {
          container: [
          // adding some basic Quill content features
          ['bold', 'italic', 'underline', 'strike'],        // toggled buttons
          ['blockquote', 'code-block'],
          ['link', 'image', 'video', 'formula'],

          [{ 'header': 1 }, { 'header': 2 }],               // custom button values
          [{ 'list': 'ordered' }, { 'list': 'bullet' }, { 'list': 'check' }],
          [{ 'script': 'sub' }, { 'script': 'super' }],      // superscript/subscript
          [{ 'indent': '-1' }, { 'indent': '+1' }],          // outdent/indent
          [{ 'direction': 'rtl' }],                         // text direction

          // [{ 'size': ['small', false, 'large', 'huge'] }],  // custom dropdown
          [{ 'header': [1, 2, 3, 4, 5, 6, false] }],

          [{ 'color': [] }, { 'background': [] }],          // dropdown with defaults from theme
          // [{ 'font': [] }],
          [{ 'align': [] }],
          [
      { 'table-body': 'TBODY' },
      { 'table-insert-rows': 'ITR' },
      { 'table-insert-columns': 'TIC' },
      { 'table-delete-rows': 'IDR' },
      { 'table-delete-columns': 'TDC' },
      { 'table-delete-body': 'TDB' }
    ],

          ['clean']
        ],
        handlers: {
          'table-body': function (val) {
        this.quill.getModule('table').insertTable(2, 2) // this引用工具栏实例, val表示按钮是否处于活动状态active
      },
      'table-insert-rows': function () {
        this.quill.getModule('table').insertRowBelow()
      },
      'table-insert-columns': function () {
        this.quill.getModule('table').insertColumnRight()
      },
      'table-delete-rows': function () {
        this.quill.getModule('table').deleteRow()
      },
      'table-delete-columns': function () {
       this.quill.getModule('table').deleteColumn()
      },
      'table-delete-body': function () {
        this.quill.getModule('table').deleteTable()
      }
        }
        },
        history: {
          // Local undo shouldn't undo changes
          // from remote users
          userOnly: true,
        },
        table: true,  // disable table module
      },
      placeholder: 'Start collaborating...',
      theme: 'snow',
    })


    provider = new WebsocketProvider(
      'ws://localhost:1234',
      `quill-demo-room`,
      ydoc,
    )
    binding = new QuillBinding(ytext, quillObj, provider.awareness)

    provider.awareness.setLocalStateField('user', {
      name: `用户${Math.floor(Math.random() * 100)}`,
      color: 'blue',
      avatar: '/布偶猫.svg',
      // color: 'blue'
    })

    provider.on('sync', () => {
      const list = [...(provider as WebsocketProvider).awareness.getStates().entries()].map(item => item[1])

      onlineUserList.value = list.map(item => ({
        name: item.user.name,
        avatar: item.user.avatar
      }))
    })

    ydoc.on('update', () => {

      const list = [...(provider as WebsocketProvider).awareness.getStates().entries()].map(item => item[1])
      onlineUserList.value = list.map(item => ({
        name: item.user.name,
        avatar: item.user.avatar
      }))
    })
  }
})


onUnmounted(() => {
  provider?.disconnect()
  quillObj = null
  // console.log(quillObj)
  // quillObj?.disable?.()
  // quillObj = null
})

</script>

<style scoped>
.read-the-docs {
  color: #888;
}

.userLine {
  display: flex;
  justify-content: flex-end;
}

.avatarImg {
  width: 25px;
  height: 25px;
  border-radius: 50%;
}
</style>
