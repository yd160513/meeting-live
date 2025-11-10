<!-- src/App.vue -->
<script setup lang="ts">
import { RouterView } from 'vue-router'
import { onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

// 检查是否应该重定向到登录页
onMounted(() => {
  // 如果当前路径不是登录页且没有必要的查询参数，则重定向到登录页
  // if (route.name !== 'login' && !route.query.userId) {
  //   router.push('/login')
  // }
})
</script>

<template>
  <router-view></router-view>
</template>

<style scoped>
/* 保留原有样式 */
</style>
