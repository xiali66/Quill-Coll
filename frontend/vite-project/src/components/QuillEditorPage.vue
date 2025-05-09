<template>
  <div class="userLine" :onLoad="loadedLine">
    <img class="avatarImg" :src="item.avatar" :title="item.name" v-for="(item, key) of onlineUserList" :key="key" />
  </div>
  <div ref="quillRef"></div>

  <div ref="viewerRef"></div>

  <div>
    <div v-if="commentId">评论{{ commentId }}</div>
    <div v-if="commentId">
      <button @click="cancelComment">
        取消
      </button>
      <button @click="confirmComment">
        创建
      </button>
    </div>
    <button @click="getDeltaData">
      获取Delta数据
    </button>
    
    <button @click="showPreview">预览</button>
  </div>
</template>

<script setup lang="ts">
import Quill from 'quill'
import QuillCursors from 'quill-cursors'
import 'quill/dist/quill.snow.css'
import { onMounted, onUnmounted, ref } from 'vue'
import * as Y from 'yjs'
import { QuillBinding } from 'y-quill'
import { WebsocketProvider } from 'y-websocket'
import 'quill-better-table/dist/quill-better-table.css'
import { tableSvgNameToIconMap, toolbarNameToTitleMap } from './utils.ts'
import { CustomComment } from './CustomCommentBlot.ts'
import katex from 'katex'
import QuillResize from 'quill-resize-module'
import QuillImageDropAndPaste  from 'quill-image-drop-and-paste'


Quill.register('modules/resize', QuillResize)
Quill.register('modules/imageDropAndPaste', QuillImageDropAndPaste)

let quillObj = null as null | Quill
let viewerObj = null as null | Quill
const quillRef = ref<HTMLElement>()
const viewerRef = ref<HTMLElement>()
let provider = null as null | WebsocketProvider

Quill.register('modules/cursors', QuillCursors)
Quill.register(CustomComment);
const ydoc = new Y.Doc()
// 在文档上定义共享文本类型
const ytext = ydoc.getText('quill')

const onlineUserList = ref<any[]>([])

const commentId = ref('')

// TableEmbed.register()

// const embeds = {
//   'table-embed': tableEmbed
// }

// 创建一个编辑器绑定 将quill编辑器“绑定”到 Y.Text 类型。
let binding = null
let firstConnect = true

// @ts-ignore
window.katex = katex

onMounted(() => {
  if (quillRef.value) {

    viewerObj = new Quill(viewerRef.value, {
      modules: {
        theme: 'snow'
      }
    })

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
              { 'table': 'TBODY' },
              { 'table-insert-rows': 'ITR' },
              { 'table-insert-columns': 'TIC' },
              { 'table-delete-rows': 'IDR' },
              { 'table-delete-columns': 'TDC' },
              { 'table-delete-body': 'TDB' },
              'custom-comment',
            ],

            ['clean']
          ],
          handlers: {
            'table': function () {
              // @ts-ignore
              this.quill.getModule('table').insertTable(2, 2) // this引用工具栏实例, val表示按钮是否处于活动状态active
            },
            'table-insert-rows': function () {
              // @ts-ignore
              this.quill.getModule('table').insertRowBelow()
            },
            'table-insert-columns': function () {
              // @ts-ignore
              this.quill.getModule('table').insertColumnRight()
            },
            'table-delete-rows': function () {
              // @ts-ignore
              this.quill.getModule('table').deleteRow()
            },
            'table-delete-columns': function () {
              // @ts-ignore
              this.quill.getModule('table').deleteColumn()
            },
            'table-delete-body': function () {
              // @ts-ignore
              this.quill.getModule('table').deleteTable()
            },
            'custom-comment': function () {
              // @ts-ignore
              const range = this.quill.getSelection();
              if (!isAddComment()) {
                return
              }
              if (range && range.length > 0) {
                if (commentId.value) {
                  const commentNodes = document.querySelectorAll(`span[data-custom-comment="${commentId.value}"]`)
                  for (const node of commentNodes) {
                    node.removeAttribute('data-custom-comment')
                  }
                }
                let id = Date.now().toString()
                commentId.value = id
                // @ts-ignore
                this.quill.formatText(range.index, range.length, 'custom-comment', id);
              }
            },
            'image':
              function imageHandler() {
                const input = document.createElement('input');
                input.setAttribute('type', 'file');
                input.setAttribute('accept', 'image/*');
                input.click();

                input.onchange = () => {
                  const file = input.files[0];
                  console.log(file)
                  const range = this.quill.getSelection();
                  this.quill.insertEmbed(range.index, 'image', 'https://pica.zhimg.com/v2-a89d7a5e0a4464ebd627a514f3f76b3a_720w.jpg?source=172ae18b')

                  // 将图片上传到服务器
                };
              }
          }
        },
        history: {
          // Local undo shouldn't undo changes
          // from remote users
          userOnly: true,
        },
        table: true,  // disable table module
        imageDropAndPaste: {
          handler: function imageHandler(imageDataUrl, type, imageData) {
            const range = this.quill.getSelection()
            this.quill.insertEmbed(range.index, 'image', 'https://pica.zhimg.com/v2-a89d7a5e0a4464ebd627a514f3f76b3a_720w.jpg?source=172ae18b')
          }
        },
        resize: {
          modules: [QuillResize.Modules.Resize, 'DisplaySize']
        }
      },
      placeholder: 'Start collaborating...',
      theme: 'snow',
    })

    const host = window.location.host.split(':')[0]
    provider = new WebsocketProvider(
      `ws://${host}:1234`,
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

    // provider.on('status', data => {
    //   console.log(data)
    //   console.log('dddd=>',  [...(provider as WebsocketProvider).awareness.getStates().entries()])
    //   if (data.status === 'connected' && (provider as WebsocketProvider).awareness.getStates().size === 1) {
    //     quillObj?.setContents([
    //       { insert: 'Hello ' },
    //       { insert: 'World!', attributes: { bold: true } },
    //       { insert: '\n' }
    //     ])
    //   }
    // })

    provider.on('sync', () => {
      const list = [...(provider as WebsocketProvider).awareness.getStates().entries()].map(item => item[1])

      if (firstConnect && list.length === 1) {
        quillObj?.setContents([
          { insert: 'Hello ' },
          { insert: 'World!', attributes: { bold: true } },
          { insert: '\n' }
        ])
        firstConnect = false
      }
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

    for (const key of tableSvgNameToIconMap.keys()) {
      const dom = document.querySelector(`.ql-${key}`)
      if (dom) {
        dom.innerHTML = tableSvgNameToIconMap.get(key) as string
      }
    }

    for (const key of toolbarNameToTitleMap.keys()) {
      const dom = document.querySelector(`.ql-${key}`)
      if (dom) {
        (dom as HTMLButtonElement).title = toolbarNameToTitleMap.get(key) as string
      }
    }

    // const insertRowSvg = ''
    // const dom = document.querySelector('.ql-table-insert-rows')
    // dom.innerHTML = insertRowSvg

    // setTimeout(() => {
    //   quillObj.setContents([
    //   { insert: 'Hello ' },
    //   { insert: 'World!', attributes: { bold: true } },
    //   { insert: '\n' }
    // ])
    // }, 0);
  }
})


onUnmounted(() => {
  provider?.disconnect()
  quillObj = null
  // console.log(quillObj)
  // quillObj?.disable?.()
  // quillObj = null
})

function isAddComment() {
  const range = quillObj?.getSelection()
  if (!range || range.length === 0) return []

  let index = range.index + 1
  const end = range.index + range.length
  let flag = true

  const [leaf, offset] = quillObj.getLeaf(index)
  if (leaf.length() === offset) {
    index += 1
  }
  while (index < end) {
    const [leaf, offset] = quillObj.getLeaf(index);
    // console.log(leaf.)
    if (leaf.parent.statics.blotName === 'custom-comment') {
      flag = false
      break
    }

    // 移动到下一个叶子节点的起始位置
    const increse = leaf.length() - offset
    index += increse === 0 ? 1 : increse;
  }

  return flag;
}

function getDeltaData() {
  console.log(quillObj?.getContents())
}

function cancelComment() {
  const commentNodes = document.querySelectorAll(`span[data-custom-comment="${commentId.value}"]`)
  for (const node of commentNodes) {
    node.removeAttribute('data-custom-comment')
  }
  commentId.value = ''
  quillObj?.setSelection(null)
}

function confirmComment() {
  commentId.value = ''
  quillObj?.setSelection(null)
}

function loadedLine() {
  console.log('111111')
}

function showPreview() {
  viewerObj?.setContents(quillObj?.getContents())
}
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

<style>
.ql-snow.ql-toolbar button svg {
  width: 18px;
}

.ql-snow.ql-toolbar {
  text-align: left;
}

span[data-custom-comment] {
  background-color: rgb(175, 200, 233);
}

span[data-custom-comment="v-bind(commentId)"] {
  background-color: blue;
}
</style>
