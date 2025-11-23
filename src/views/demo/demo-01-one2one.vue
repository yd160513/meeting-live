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
                @click="initiateCall(participant.userId)"
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
          <video ref="remoteVideo" autoplay playsinline class="remote-video"></video>
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
import { ref, onMounted, onUnmounted } from 'vue'
import { useRoute } from 'vue-router'
import { io } from "socket.io-client"
import { message } from 'ant-design-vue'

// 获取URL参数: // 获取URL参数: https://localhost:8080/?userId=111&roomId=10086
const route = useRoute()
const currentUserId = route.query.userId as string
const roomId = route.query.roomId as string

// WebRTC 相关引用
const localVideo = ref<HTMLVideoElement | null>(null)
const remoteVideo = ref<HTMLVideoElement | null>(null)

// 状态管理
const isCalling = ref(false)
const callingTarget = ref<string | null>(null)
const activeRoomId = ref<string | null>(null)
const participants = ref<Array<{ userId: string }>>([])
const localMessage = ref('')
const remoteMessage = ref('')
const chatHistory = ref<HTMLElement | null>(null)

// WebRTC 对象
let pc: RTCPeerConnection | null = null
let chatChannel: RTCDataChannel | null = null
let localStream: MediaStream | null = null

// Socket 连接
const socket = io('', {
  query: {
    userId: currentUserId,
    roomId: roomId
  }
})

// Socket 事件监听
socket.on('connect', () => {
  console.log('Socket connected')
  joinRoom()
})

socket.on('disconnect', () => {
  console.log("Socket disconnected")
})

socket.on('connect_error', (error) => {
  console.error("Socket connection error: ", error)
  message.error('连接服务器失败')
})

socket.on("message", (data) => {
  console.log('收到消息: ', data)
  handleMessage(data)
})

// 消息处理中心
const handleMessage = (data: any) => {
  switch (data.type) {
    case 'join_self':
      message.info('成功加入房间！')
      break
    case 'room_users':
      participants.value = data.data
      break
    case 'call':
      handleIncomingCall(data.data)
      break
    case 'hangup':
      hangup(true)
      break
    case 'offer':
      handleOffer(data.data)
      break
    case 'answer':
      handleAnswer(data.data)
      break
    case 'candidate':
      handleCandidate(data.data)
      break
  }
}

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

// 加入房间
const joinRoom = () => {
  socket.emit('message', buildMessage('join', currentUserId, roomId))
}

// 初始化 WebRTC 连接
const initWebRTC = async (isCaller: boolean, targetUserId: string) => {
  try {
    // 创建 RTCPeerConnection
    pc = new RTCPeerConnection({
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
        { urls: 'stun:stun1.l.google.com:19302' }
      ]
    })

    // 设置事件处理器
    setupPeerConnectionEvents(isCaller, targetUserId)

    // 获取本地媒体流
    localStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: true
    })

    // 将本地流绑定到视频元素
    if (localVideo.value) {
      localVideo.value.srcObject = localStream
    }

    // 添加轨道到连接
    localStream.getTracks().forEach(track => {
      pc!.addTrack(track, localStream!)
    })

    return pc
  } catch (error) {
    console.error('获取媒体流失败:', error)
    message.error('无法访问摄像头或麦克风')
    return null
  }
}

// 设置 PeerConnection 事件处理器
const setupPeerConnectionEvents = (isCaller: boolean, targetUserId: string) => {
  if (!pc) return

  // 接收远程流
  pc.ontrack = (event) => {
    console.log(`${isCaller ? '主叫' : '被叫'}收到 ontrack:`, event)
    if (remoteVideo.value) {
      remoteVideo.value.srcObject = event.streams[0]!
    }
  }

  // ICE 连接状态变化
  pc.oniceconnectionstatechange = () => {
    console.log('ICE连接状态变化:', pc!.iceConnectionState)
    if (pc!.iceConnectionState === 'connected' || pc!.iceConnectionState === 'completed') {
      console.log('WebRTC协商成功！')
      message.success('连接已建立')
    } else if (pc!.iceConnectionState === 'failed') {
      console.log('WebRTC连接失败')
      message.error('连接失败')
      hangup()
    }
  }

  // 连接状态变化
  pc.onconnectionstatechange = () => {
    console.log('连接状态变化:', pc!.connectionState)
    if (pc!.connectionState === 'connected') {
      console.log('WebRTC连接已建立')
    } else if (pc!.connectionState === 'disconnected' || pc!.connectionState === 'failed') {
      hangup()
    }
  }

  // ICE 候选者处理
  pc.onicecandidate = (event) => {
    if (event.candidate) {
      socket.emit('message', buildMessage(
          'candidate',
          currentUserId,
          activeRoomId.value!,
          targetUserId,
          event.candidate
      ))
    }
  }

  // 数据通道处理
  if (isCaller) {
    chatChannel = pc.createDataChannel('chat')
    setupDataChannel(chatChannel)
  } else {
    pc.ondatachannel = (event) => {
      chatChannel = event.channel
      setupDataChannel(chatChannel)
    }
  }
}

// 设置数据通道
const setupDataChannel = (channel: RTCDataChannel) => {
  channel.onmessage = (event) => {
    console.log('收到数据消息:', event)
    remoteMessage.value = event.data
  }

  channel.onopen = () => {
    console.log('数据通道已打开')
  }

  channel.onclose = () => {
    console.log('数据通道已关闭')
  }

  channel.onerror = (error) => {
    console.error('数据通道错误:', error)
  }
}

// 发起呼叫
const initiateCall = async (targetUserId: string) => {
  console.log(`正在呼叫用户 ${targetUserId}`)

  // 通知被叫方
  socket.emit('message', buildMessage(
      'call',
      currentUserId,
      roomId,
      targetUserId
  ))

  // 初始化 WebRTC
  const peerConnection = await initWebRTC(true, targetUserId)
  if (!peerConnection) return

  // 更新状态
  callingTarget.value = targetUserId
  activeRoomId.value = roomId
  isCalling.value = true

  // 创建 Offer
  try {
    const offer = await peerConnection.createOffer()
    await peerConnection.setLocalDescription(offer)

    // 发送 Offer
    socket.emit('message', buildMessage(
        'offer',
        currentUserId,
        roomId,
        targetUserId,
        offer
    ))
  } catch (error) {
    console.error('创建 Offer 失败:', error)
    message.error('呼叫失败')
    hangup()
  }
}

// 处理来电
const handleIncomingCall = async (data: any) => {
  console.log('收到呼叫:', data)
  message.info(`${data.userId} 来电，即将自动接听！`)

  // 初始化 WebRTC
  const peerConnection = await initWebRTC(false, data.userId)
  if (!peerConnection) return

  // 更新状态
  callingTarget.value = data.userId
  activeRoomId.value = data.roomId
  isCalling.value = true
}

// 处理 Offer
const handleOffer = async (data: any) => {
  console.log('收到 Offer:', data)
  if (!pc) return

  try {
    await pc.setRemoteDescription(data.sdpObj)
    const answer = await pc.createAnswer()
    await pc.setLocalDescription(answer)

    socket.emit('message', buildMessage(
        'answer',
        currentUserId,
        data.roomId,
        data.userId,
        answer
    ))
  } catch (error) {
    console.error('处理 Offer 失败:', error)
    message.error('连接建立失败')
    hangup()
  }
}

// 处理 Answer
const handleAnswer = async (data: any) => {
  console.log('收到 Answer:', data)
  if (!pc) return

  try {
    await pc.setRemoteDescription(data.sdpObj)
  } catch (error) {
    console.error('处理 Answer 失败:', error)
    message.error('连接建立失败')
    hangup()
  }
}

// 处理 ICE 候选者
const handleCandidate = async (data: any) => {
  if (!pc) return

  try {
    await pc.addIceCandidate(data.sdpObj)
  } catch (error) {
    console.error('添加 ICE 候选者失败:', error)
  }
}

// 挂断通话
const hangup = (isRemote = false) => {
  console.log('准备断开连接...')

  // 停止本地媒体流
  if (localStream) {
    localStream.getTracks().forEach(track => track.stop())
    localStream = null
  }

  // 清理本地视频元素
  if (localVideo.value) {
    localVideo.value.srcObject = null
  }

  // 清理远程视频元素
  if (remoteVideo.value) {
    remoteVideo.value.srcObject = null
  }

  // 关闭数据通道
  if (chatChannel) {
    chatChannel.close()
    chatChannel = null
  }

  // 关闭 PeerConnection
  if (pc) {
    pc.close()
    pc = null
  }

  // 通知对方（如果不是远程挂断）
  if (!isRemote && activeRoomId.value && callingTarget.value) {
    socket.emit('message', buildMessage(
        'hangup',
        currentUserId,
        activeRoomId.value,
        callingTarget.value
    ))
  }

  // 重置状态
  isCalling.value = false
  callingTarget.value = null
  activeRoomId.value = null

  // 显示提示信息
  if (isRemote) {
    message.info('对方已挂断通话')
  } else {
    message.info('通话已结束')
  }

  console.log('连接已关闭')
}

// 发送消息
const sendMessage = () => {
  const msg = localMessage.value.trim()
  if (!msg) return

  if (chatChannel && chatChannel.readyState === 'open') {
    chatChannel.send(msg)
    localMessage.value = ''
  } else {
    message.warning('数据通道未就绪，无法发送消息')
  }
}

// 组件卸载时清理资源
onUnmounted(() => {
  hangup()
  socket.disconnect()
})

// 初始化
onMounted(() => {
  console.log('初始化一对一通话组件...')
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

.call-button {
  padding: 6px 12px;
  background-color: #409eff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.call-button:hover {
  background-color: #337ecc;
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
