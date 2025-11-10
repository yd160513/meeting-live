<!-- src/views/LoginView.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { Form, Input, Button } from 'ant-design-vue'

const router = useRouter()

// 表单数据
const formState = ref({
  userId: '',
  roomId: '',
  targetId: ''
})

// 组件挂载时从URL参数中读取值
onMounted(() => {
  const urlParams = new URLSearchParams(window.location.search)
  formState.value.userId = urlParams.get('userId') || ''
  formState.value.roomId = urlParams.get('roomId') || ''
  formState.value.targetId = urlParams.get('targetId') || ''
})

// 登录处理
const handleLogin = () => {
  if (!formState.value.userId || !formState.value.roomId) {
    return
  }

  // 跳转到会议页面，携带参数
  router.push({
    path: '/',
    query: {
      userId: formState.value.userId,
      roomId: formState.value.roomId,
      targetId: formState.value.targetId
    }
  })
}
</script>

<template>
  <div class="login-container">
    <div class="login-form">
      <h2>腾讯会议</h2>
      <Form
          :model="formState"
          name="login"
          class="login-form-content"
          @finish="handleLogin"
      >
        <Form.Item
            label="用户ID"
            name="userId"
            :rules="[{ required: true, message: '请输入用户ID!' }]"
        >
          <Input v-model:value="formState.userId" placeholder="请输入用户ID" />
        </Form.Item>

        <Form.Item
            label="房间号"
            name="roomId"
            :rules="[{ required: true, message: '请输入房间号!' }]"
        >
          <Input v-model:value="formState.roomId" placeholder="请输入房间号" />
        </Form.Item>

        <Form.Item
            label="目标用户ID"
            name="targetId"
        >
          <Input v-model:value="formState.targetId" placeholder="请输入要呼叫的用户ID(可选)" />
        </Form.Item>

        <Form.Item>
          <Button type="primary" html-type="submit" block>
            进入会议
          </Button>
        </Form.Item>
      </Form>
    </div>
  </div>
</template>

<style scoped>
.login-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: #f0f2f5;
}

.login-form {
  width: 400px;
  padding: 40px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.login-form h2 {
  text-align: center;
  margin-bottom: 30px;
  color: #1890ff;
}

.login-form-content {
  max-width: 300px;
  margin: 0 auto;
}
</style>
