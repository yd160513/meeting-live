<script setup lang="ts">
import {io} from "socket.io-client";
import {useRoute} from "vue-router";
import {onUnmounted} from "vue";

const route = useRoute()
const userId = route.query.userId || ''
const roomId = route.query.roomId || ''
console.log(userId, roomId)
const socket = io("", {
  query: {
    userId,
    roomId
  }
});

socket.on("connect", async () => {
  console.log("socket connected");

  joinHandle()
});
socket.on("disconnect", () => {
  console.log("socket disconnected");
})

socket.on('message', (data) => {
  console.log('message: ', data)
})

// 加入房间
const joinHandle = () => {
  socket.emit('message', {type: 'join', data: {userId, roomId}})
}

// 离开房间
const leaveHandle = () => {
  socket.emit('message', {type: 'leave', data: {userId, roomId}})
}

onUnmounted(() => {
  socket.disconnect()
})
</script>

<template>
  <main>
    home-view

  <button @click="leaveHandle">离开房间</button>
  </main>
</template>
