<template>
  <div class="one2one-container">
    <!-- 左侧人员列表 -->
    <div class="left-panel">
      <div class="participants-list">
        <div
            v-for="participant in participants"
            :key="participant.userId"
            class="participant-item"
        >
          <div class="participant-info">
            <div class="avatar">{{ participant.userId }}</div>
            <div class="name">{{ participant.userId }}</div>
          </div>
          <div class="call-buttons">
            <button
                v-if="participant.userId !== currentUserId && !isCalling"
                class="call-button"
                @click="callParticipant(participant.userId)"
            >
              通话
            </button>
            <button
                v-if="isCalling && callingTarget === participant.userId"
                class="hangup-button"
                @click="hangup(false)"
            >
              挂断
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- 中间视频区域 -->
    <div class="center-panel">
      <div class="video-container">
        <div class="local-video-wrapper">
          <video ref="localVideo" autoplay playsinline muted class="local-video"></video>
          <div class="video-label">我</div>
        </div>
        <div class="remote-video-wrapper">
          <video ref="remoteVideo" autoplay playsinline muted class="remote-video"></video>
          <div class="video-label">对方</div>
        </div>
      </div>
    </div>

    <!-- 右侧聊天区域 -->
    <div class="right-panel">
      <div class="chat-container">
        <!-- 聊天历史记录 -->
        <div class="chat-history" ref="chatHistory">
          <div class="remote-message">收到消息: {{ remoteMessage }}</div>
        </div>

        <!-- 消息输入区域 -->
        <div class="chat-input-area">
          <input
              v-model="localMessage"
              type="text"
              placeholder="输入消息..."
              class="message-input"
              @keyup.enter="sendMessage"
          />
          <button @click="sendMessage" class="send-button">发送</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {ref, onMounted, nextTick, h} from 'vue'
import {useRoute} from 'vue-router'
import {io} from "socket.io-client";
import {message, notification, Button} from 'ant-design-vue';

// 获取URL参数: https://localhost:8080/?userId=111&roomId=10086&targetId=222
const route = useRoute()
const currentUserId = route.query.userId as string
const roomId = route.query.roomId as string
const targetId = route.query.targetId as string

let pc: RTCPeerConnection | null
const localVideo = ref<HTMLVideoElement | null>(null)
const remoteVideo = ref<HTMLVideoElement | null>(null)
let chatChannel: RTCDataChannel | null

// 添加状态管理变量
const isCalling = ref(false)
const callingTarget = ref<string | null>(null)
const activeRoomId = ref<string | null>(null)

const socket = io('', {
  query: {
    userId: currentUserId,
    roomId: roomId
  }
})

socket.on('connect', () => {
  console.log('socket connected');

  joinHandle()
})

socket.on('disconnect', () => {
  console.log("socket disconnected");
})

socket.on('connect_error', (error) => {
  console.log("socket error: ", error);
})

const handleAnswer = async (data: any) => {
  console.log('收到answer: ', data)
  if (!pc) return
  await pc.setRemoteDescription(data.sdpObj)
}

const handleOffer = async (data: any) => {
  console.log('========收到offer: ', data)
  if (!pc) return
  await pc.setRemoteDescription(data.sdpObj)
  const answer = await pc.createAnswer()
  await pc.setLocalDescription(answer)
  socket.emit('message', buildMessage('answer', currentUserId, data.roomId, data.userId, answer))

  // 设置呼叫状态
  isCalling.value = true
  callingTarget.value = data.userId
  activeRoomId.value = data.roomId
}

socket.on("message", (data) => {
  console.log('收到消息: ', data)
  switch (data.type) {
    case 'join_self':
      message.info('恭喜你！成功加入房间！');
      break;
    case 'room_users':
      participants.value = data.data
      break;
    case 'call':
      message.info(`${data.data.userId} 来电, 即将自动接听！`);
      onCallHandle(data.data)
      break;
    case 'hangup':
      hangup(true)
      break;
    case 'offer':
      handleOffer(data.data)
      break;
    case 'answer':
      handleAnswer(data.data)
      break;
    case 'candidate':
      handleCandidate(data.data)
      break;
  }
});

// 构建消息体
const buildMessage = (type: string, userId: string, roomId: string, targetId?: string, sdpObj?: any) => {
  return {
    type,
    data: {
      userId,
      roomId,
      targetId,
      sdpObj
    }
  }
}

const handleCandidate = async (data) => {
  if (!pc) return
  await pc.addIceCandidate(data.sdpObj)
}

// 加入房间
const joinHandle = () => {
  socket.emit('message', buildMessage('join', currentUserId, roomId))
}

const setupDataChannel = (channel: RTCDataChannel) => {
  channel.onmessage = (event) => {
    console.log('channel.onmessage: ', event)
    remoteMessage.value = event.data
  }
  channel.onopen = (e) => {
    console.log('channel.onopen', e)
  }
  channel.onclose = (e) => {
    console.log('channel.onclose', e)
  }
}

const callerHandle = async () => {
  pc = new RTCPeerConnection()
  if (!pc) return
  pc.ontrack = (event) => {
    // 将远端流绑定到 video 元素
    console.log('主叫收到 ontrack: ', event)
    remoteVideo.value!.srcObject = event.streams[0]!
  }
  pc.oniceconnectionstatechange = () => {
    if (!pc) return
    console.log('ICE连接状态变化:', pc.iceConnectionState);
    if (pc.iceConnectionState === 'connected' || pc.iceConnectionState === 'completed') {
      console.log('WebRTC协商成功！');
    } else if (pc.iceConnectionState === 'failed') {
      console.log('WebRTC连接失败');
    }
  };
  pc.onconnectionstatechange = () => {
    if (!pc) return
    console.log('连接状态变化:', pc.connectionState);
    if (pc.connectionState === 'connected') {
      console.log('WebRTC连接已建立');
    }
  };
  pc.onicecandidate = (event) => {
    console.log('onicecandidate: ', event)
    // 和对方交换 ice
    if (event.candidate) {
      socket.emit('message', buildMessage('candidate', currentUserId, roomId, targetId, event.candidate))
    }
  }
  chatChannel = pc.createDataChannel('chat')

  setupDataChannel(chatChannel)

  // 获取本地音视频流
  const stream = await navigator.mediaDevices.getUserMedia({video: true, audio: true})
  stream.getTracks().forEach(track => {
    if (!pc) return
    pc.addTrack(track, stream)
  })
  // 创建 offer
  const offer = await pc.createOffer()
  console.log('=====', offer.sdp)
  await pc.setLocalDescription(offer)
  // 将本地音视频流绑定到 video 元素
  if (!localVideo.value) {
    console.error('localVideo is null')
    return
  }
  localVideo.value.srcObject = stream
  console.log('====')
  socket.emit('message', buildMessage('offer', currentUserId, roomId, targetId, offer))
}

// 收到呼叫
const onCallHandle = async (data: any) => {
  console.log('====', data)
  pc = new RTCPeerConnection()
  pc.ontrack = (event) => {
    console.log('被叫收到 ontrack: ', event)
    remoteVideo.value!.srcObject = event.streams[0]!
  }
  pc.oniceconnectionstatechange = () => {
    if (!pc) return
    console.log('ICE连接状态变化:', pc.iceConnectionState);
    if (pc.iceConnectionState === 'connected' || pc.iceConnectionState === 'completed') {
      console.log('WebRTC协商成功！');
      // 设置呼叫状态
      isCalling.value = true
      callingTarget.value = data.userId
      activeRoomId.value = data.roomId
    } else if (pc.iceConnectionState === 'failed') {
      console.log('WebRTC连接失败');
    }
  };
  pc.onconnectionstatechange = () => {
    if (!pc) return
    console.log('连接状态变化:', pc.connectionState);
    if (pc.connectionState === 'connected') {
      console.log('WebRTC连接已建立');
    }
  };
  pc.onicecandidate = (event) => {
    console.log('被叫收到 onicecandidate: ', event)
    // 和对方交换 ice
    if (event.candidate) {
      socket.emit('message', buildMessage('candidate', currentUserId, data.roomId, data.userId, event.candidate))
    }
  }

  pc.ondatachannel = (event) => {
    console.log('onicecandidate: ', event)
    chatChannel = event.channel
    setupDataChannel(chatChannel)
  };
  console.log('========收到呼叫准备获取本地流')
  const stream = await navigator.mediaDevices.getUserMedia({video: true, audio: true})
  console.log('========收到呼叫，准备执行 addTrack')
  stream.getTracks().forEach(track => {
    if (!pc) return
    pc.addTrack(track, stream)
  })
  if (!localVideo.value) {
    console.error('localVideo is null')
    return
  }
  localVideo.value.srcObject = stream
}

// 呼叫某人
const callParticipant = (targetId: string) => {
  console.log(`正在呼叫 ${roomId} 房间，用户 ${targetId}`)
  const message = buildMessage('call', currentUserId, roomId, targetId)

  socket.emit('message', message)
  callerHandle()

  // 更新呼叫状态
  isCalling.value = true
  callingTarget.value = targetId
  activeRoomId.value = roomId
}

const hangup = (isRemote = false) => {
  console.log('准备断开P2P连接...')

  // 1. 停止本地媒体流
  if (localVideo.value?.srcObject) {
    const tracks = (localVideo.value.srcObject as MediaStream).getTracks()
    tracks.forEach(track => track.stop())
    localVideo.value.srcObject = null
  }

  // 2. 关闭 dataChannel
  if (chatChannel) {
    chatChannel.close()
    chatChannel = null
  }

  // 3. 关闭 peerConnection
  if (pc) {
    pc.close()
    pc = null
  }

  // 4. 通知对方（可选，但建议做）
  if (!isRemote && activeRoomId.value && callingTarget.value) {
    socket.emit('message', buildMessage('hangup', currentUserId, activeRoomId.value, callingTarget.value))
  }

  // 5. 重置呼叫状态
  isCalling.value = false
  callingTarget.value = null
  activeRoomId.value = null

  // 6. UI 清理
  if (remoteVideo.value) {
    remoteVideo.value.srcObject = null
  }

  // 7. 显示提示信息
  if (isRemote) {
    message.info('对方已挂断通话');
  } else {
    message.info('通话已结束');
  }

  console.log('P2P 连接已关闭')
}

// 聊天相关
const chatHistory = ref<HTMLElement | null>(null)
const localMessage = ref('')
const remoteMessage = ref('')


// 参会人员列表
const participants = ref()


// 发送消息
const sendMessage = () => {
  if (localMessage.value.trim() === '') return

  chatChannel?.send(localMessage.value)

  localMessage.value = ''
}

// 格式化时间
const formatTime = (timestamp: number) => {
  const date = new Date(timestamp)
  return `${date.getHours().toString().padStart(2, '0')}:${date.getMinutes().toString().padStart(2, '0')}`
}

// 滚动到聊天记录底部
const scrollToBottom = () => {
  nextTick(() => {
    if (chatHistory.value) {
      chatHistory.value.scrollTop = chatHistory.value.scrollHeight
    }
  })
}

// 初始化视频
const initVideo = () => {
  // 这里可以添加实际的视频流获取逻辑
  console.log('初始化视频...')
}

onMounted(() => {
  initVideo()
  scrollToBottom()
})
</script>

<style scoped>
.one2one-container {
  display: flex;
  height: 100vh;
  padding: 20px;
  box-sizing: border-box;
  background-color: #f5f5f5;
}

.left-panel {
  width: 250px;
  background: white;
  border-radius: 8px;
  padding: 16px;
  margin-right: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.participants-list {
  height: 100%;
  overflow-y: auto;
}

.participant-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 0;
  border-bottom: 1px solid #eee;
}

.participant-info {
  display: flex;
  align-items: center;
}

.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: #409eff;
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  margin-right: 12px;
}

.name {
  font-size: 14px;
}

.call-buttons {
  display: flex;
  gap: 5px;
}

.hangup-button {
  padding: 6px 12px;
  background-color: #ff4d4f;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.hangup-button:hover {
  background-color: #d9363e;
}

.center-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  background: white;
  border-radius: 8px;
  padding: 16px;
  margin-right: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.video-container {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.local-video-wrapper,
.remote-video-wrapper {
  position: relative;
  flex: 1;
  margin-bottom: 16px;
  background-color: #333;
  border-radius: 8px;
  overflow: hidden;
}

.local-video-wrapper:last-child,
.remote-video-wrapper:last-child {
  margin-bottom: 0;
}

.local-video,
.remote-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.video-label {
  position: absolute;
  bottom: 8px;
  left: 8px;
  background-color: rgba(0, 0, 0, 0.5);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
}

.right-panel {
  width: 300px;
  display: flex;
  flex-direction: column;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.chat-container {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.chat-history {
  flex: 1;
  padding: 16px;
  overflow-y: auto;
  background-color: #fafafa;
}

.message-item {
  margin-bottom: 16px;
  max-width: 80%;
}

.message-item.sent {
  margin-left: auto;
}

.message-item.received {
  margin-right: auto;
}

.message-sender {
  font-size: 12px;
  color: #999;
  margin-bottom: 4px;
}

.message-content {
  padding: 8px 12px;
  border-radius: 8px;
  word-wrap: break-word;
}

.message-item.sent .message-content {
  background-color: #409eff;
  color: white;
  border-bottom-right-radius: 0;
}

.message-item.received .message-content {
  background-color: white;
  border: 1px solid #eee;
  border-bottom-left-radius: 0;
}

.message-time {
  font-size: 10px;
  color: #999;
  text-align: right;
  margin-top: 4px;
}

.chat-input-area {
  display: flex;
  padding: 16px;
  border-top: 1px solid #eee;
  background-color: white;
}

.message-input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  margin-right: 8px;
  outline: none;
}

.message-input:focus {
  border-color: #409eff;
}

.send-button {
  padding: 8px 16px;
  background-color: #409eff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.send-button:hover {
  background-color: #337ecc;
}
</style>
