<template>
    <div class="userLine">
        <img class="avatarImg" :src="item.avatar" :title="item.name" v-for="(item, key) of onlineUserList" :key="key" />
    </div>
    <div class="editor" ref="editorRef"></div>
</template>

<script lang="ts" setup>
import { EditorState } from "prosemirror-state"
import { EditorView } from "prosemirror-view"
import { schema } from "prosemirror-schema-basic"
import { exampleSetup } from "prosemirror-example-setup"
import { ySyncPlugin, yCursorPlugin, yUndoPlugin, undo, redo } from 'y-prosemirror'
import { keymap } from 'prosemirror-keymap'

import * as Y from 'yjs'
import { onMounted, onUnmounted, ref } from 'vue'
import './style.css'
import { WebsocketProvider } from "y-websocket"

const editorRef = ref<HTMLDivElement>()
const onlineUserList = ref<any[]>([])

let editor = null
const ydoc = new Y.Doc()
let provider = null as null | WebsocketProvider
const type = ydoc.getXmlFragment('prosemirror')

onMounted(() => {

    provider = new WebsocketProvider(
        'ws://localhost:1234',
        'test-room',
        ydoc,
    )

    provider.awareness.setLocalStateField('user', {
        name: `用户${Math.floor(Math.random() * 100)}`,
        color: 'red',
        avatar: '/布偶猫.svg',
    })

    editor = new EditorView(editorRef.value as HTMLDivElement, {
        state: EditorState.create({
            schema,
            plugins: [
                ySyncPlugin(type),
                yCursorPlugin(provider.awareness),
                yUndoPlugin(),
                keymap({
                    'Mod-z': undo,
                    'Mod-y': redo,
                    'Mod-Shift-z': redo,
                }),
            ].concat(exampleSetup({ schema }))
        })
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
})

onUnmounted(() => {
    provider?.disconnect()
    editor = null
})

</script>

<style lang="less" scoped>
.userLine {
  display: flex;
  justify-content: flex-end;
}

.avatarImg {
  width: 25px;
  height: 25px;
  border-radius: 50%;
}

.editor {
    min-height: 200px;
    border: 1px solid rgb(192, 192, 192);
}
</style>