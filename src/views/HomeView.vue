<!-- src/views/HomeView.vue -->
<script setup lang="ts">
import {ref, onUnmounted} from 'vue'
import {useRoute} from 'vue-router'
import {io} from "socket.io-client"
import {Button, Card, Row, Col} from 'ant-design-vue'

// https://localhost:8080/?userId=111&roomId=10086&targetId=222

const route = useRoute()

const roomId = (route.query.roomId || '') as string
const targetId = (route.query.targetId || '') as string
const userId = (route.query.userId || '') as string

let pc = null

// 视频流状态
const localStream = ref<MediaStream | null>(null)
const remoteStream = ref<MediaStream | null>(null)

const socketQuery = {
  userId,
  roomId
}
const socket = io("", {
  query: socketQuery
});

console.log('连接 websocket 参数: ', socketQuery)

socket.on("connect", async () => {
  console.log("socket connected");
  joinHandle()
});

socket.on("disconnect", () => {
  console.log("socket disconnected");
})

socket.on('message', async (data) => {
  console.log('客户端收到消息: ', data)
  switch (data.type) {
    case 'call':
      // 收到呼叫
      await onCallHandle(data)
      break;
    case 'offer':
      // 收到offer
      await onOfferHandle(data)
      break;
    case 'answer':
      // 收到answer
      await onAnswerHandle(data)
      break;
    case 'candidate':
      // 收到候选
      await onCandidateHandle(data)
      break;
    case 'leave':
      // 收到其他人离开
      console.log('有人离开房间: ', data.data.userId)
      break;
    case 'join':
      // 有人加入
      console.log('有人加入房间: ', data.data.userId)
      break;
    case 'room_users':
      console.log('房间用户有变化: ', data.data)
      break;
  }
})

// 构建消息
const buildMessage = (type: string, userId: string, roomId: string, targetId?: string, data:any = null) => {
  return {
    type,
    data: {
      data,
      roomId,
      userId,
      targetId,
    },
  }
}

// 收到 answer
const onAnswerHandle = async ({data}: any) => {
  console.log('onAnswerHandle: ', data)
  await pc.setRemoteDescription(data.data)
}

// 收到offer
const onOfferHandle = async ({data}: any) => {
  console.log('onOfferHandle: ', data)
  await pc.setRemoteDescription(data.data)

  const answer = await pc.createAnswer()

  await pc.setLocalDescription(answer)

  const answerData = buildMessage('answer', userId, data.roomId, data.userId, answer)
  socket.emit('message', answerData)
}

// 收到候选
const onCandidateHandle = async ({data}: any) => {
  console.log('onCandidateHandle: ', data)
  await pc.addIceCandidate(data.data)
}

// 收到呼叫
const onCallHandle = async ({data}: any) => {
  console.log('onCallHandle: ', data)
  const targetId = data.userId

  pc = new window.RTCPeerConnection()

  // 获取本地音视频流添加到 pc
  const stream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true
  })

  const localVideo = document.getElementById('local') as HTMLVideoElement
  if (localVideo) {
    localVideo.srcObject = stream
  }

  stream.getTracks().forEach(track => {
    pc.addTrack(track, stream)
  })

  pc.oniceconnectionstatechange = () => {
    console.log('ICE连接状态变化:', pc.iceConnectionState);
    if (pc.iceConnectionState === 'connected' || pc.iceConnectionState === 'completed') {
      console.log('WebRTC协商成功！');
    } else if (pc.iceConnectionState === 'failed') {
      console.log('WebRTC连接失败');
    }
  };

  pc.onconnectionstatechange = () => {
    console.log('连接状态变化:', pc.connectionState);
    if (pc.connectionState === 'connected') {
      console.log('WebRTC连接已建立');
    }
  };

  pc.onicecandidate = (event) => {
    if (event.candidate) {
      const candidateData = buildMessage('candidate', userId, roomId, targetId, event.candidate)

      socket.emit('message', candidateData)
    } else {
      console.log("在此次协商中，没有更多的候选了")
    }
  }

  // 设置远程流监听
  pc.ontrack = (event) => {
    console.warn('onCall ontrack: ', event)

    remoteStream.value = event.track
    const remoteVideo = document.getElementById('remote') as HTMLVideoElement
    console.log('remoteVideo: ', remoteVideo)
    const stream = remoteVideo.srcObject
    if (stream) {
      stream.addTrack(event.track)
    } else {
      const newStream = new MediaStream()
      newStream.addTrack(event.track)
      remoteVideo.srcObject = newStream
      remoteVideo.muted = true
    }
  }

  console.log('RTCPeerConnection 创建完成:', pc);
  console.log('初始 ICE 连接状态:', pc.iceConnectionState);
  console.log('初始连接状态:', pc.connectionState);
}

// 呼叫
const callHandle = async () => {
  if (!targetId) {
    console.log('请选择一个用户')
    return
  }

  const callData = buildMessage('call', userId, roomId, targetId)
  console.log('===== call', callData)

  socket.emit('message', callData)

  pc = new window.RTCPeerConnection()

  // 获取本地音视频流添加到 pc
  const stream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true
  })

  const localVideo = document.getElementById('local') as HTMLVideoElement
  if (localVideo) {
    localVideo.srcObject = stream
  }

  stream.getTracks().forEach(track => {
    pc.addTrack(track, stream)
  })

  pc.oniceconnectionstatechange = () => {
    console.log('ICE连接状态变化:', pc.iceConnectionState);
    if (pc.iceConnectionState === 'connected' || pc.iceConnectionState === 'completed') {
      console.log('WebRTC协商成功！');
    } else if (pc.iceConnectionState === 'failed') {
      console.log('WebRTC连接失败');
    }
  };

  pc.onconnectionstatechange = () => {
    console.log('连接状态变化:', pc.connectionState);
    if (pc.connectionState === 'connected') {
      console.log('WebRTC连接已建立');
    }
  };

  pc.onicecandidate = (event) => {
    if (event.candidate) {
      const candidateData = buildMessage('candidate', userId, roomId, targetId, event.candidate)
      console.log('call - candidate: ', candidateData)
      socket.emit('message', candidateData)
    } else {
      console.log("在此次协商中，没有更多的候选了")
    }
  }

  // 设置远程流监听
  pc.ontrack = (event) => {
    console.warn('call ontrack: ', event)
    remoteStream.value = event.track
    const remoteVideo = document.getElementById('remote') as HTMLVideoElement
    console.log('remoteVideo: ', remoteVideo)
    const stream = remoteVideo.srcObject
    if (stream) {
      stream.addTrack(event.track)
    } else {
      const newStream = new MediaStream()
      newStream.addTrack(event.track)
      remoteVideo.srcObject = newStream
      remoteVideo.muted = true
    }
  }

  console.log('RTCPeerConnection 创建完成111:', pc);
  console.log('初始 ICE 连接状态:', pc.iceConnectionState);
  console.log('初始连接状态:', pc.connectionState);

  const offer = await pc.createOffer()

  await pc.setLocalDescription(offer)

  const offerData = buildMessage('offer', userId, roomId, targetId, offer)
  console.log('====== offer', offerData)

  socket.emit('message', offerData)
}

// 加入房间
const joinHandle = () => {
  const userId = (route.query.userId || '') as string

  const joinData = buildMessage('join', userId, roomId)
  socket.emit('message', joinData)
}

// 离开房间
const leaveHandle = () => {
  const userId = (route.query.userId || '') as string

  const leaveData = buildMessage('leave', userId, roomId)
  socket.emit('message', leaveData)
}

onUnmounted(() => {
  socket.disconnect()
  // 停止媒体流
  // if (localStream.value) {
  //   localStream.value.getTracks().forEach(track => track.stop())
  // }
  // if (remoteStream.value) {
  //   remoteStream.value.getTracks().forEach(track => track.stop())
  // }
})
</script>

<template>
  <div class="meeting-room">
    <Card title="视频会议" style="width: 100%; height: 100vh;">
      <template #extra>
        <Button type="primary" danger @click="leaveHandle">离开房间</Button>
      </template>

      <Row :gutter="16">
        <Col :span="12">
          <div class="video-container">
            <h3>本地视频</h3>
            <video autoplay playsinline muted id="local" class="video-stream"></video>
          </div>
        </Col>
        <Col :span="12">
          <div class="video-container">
            <h3>远程视频</h3>
            <video autoplay playsinline id="remote" class="video-stream"></video>
          </div>
        </Col>
      </Row>

      <a-button>test</a-button>

      <div class="control-panel">
        <Button
            type="primary"
            size="large"
            @click="callHandle"
            :disabled="!targetId"
        >
          呼叫用户
        </Button>
        <div class="room-info">
          <p>用户ID: {{ userId }}</p>
          <p>房间号: {{ roomId }}</p>
          <p v-if="targetId">目标用户: {{ targetId }}</p>
        </div>
      </div>
    </Card>
  </div>
</template>

<style scoped>
.meeting-room {
  height: 100vh;
  background-color: #f0f2f5;
  padding: 20px;
  box-sizing: border-box;
}

.video-container {
  text-align: center;
  margin-bottom: 20px;
}

.video-container h3 {
  margin-bottom: 10px;
  color: #333;
}

.video-stream {
  width: 100%;
  max-width: 600px;
  height: auto;
  background-color: #000;
  border-radius: 8px;
}

.control-panel {
  text-align: center;
  margin-top: 20px;
}

.room-info {
  margin-top: 20px;
  padding: 15px;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}
</style>
