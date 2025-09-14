<template>
  <div class="chat-container">
    <div class="messages" ref="messagesContainer">
      <div v-for="message in messages" :key="message.id" 
           :class="['message', message.type]">
        <div class="content">{{ message.content }}</div>
        <div v-if="message.showOptions" class="options">
          <button v-for="option in message.options" :key="option" 
                  @click="selectOption(option)" class="option-btn">
            {{ option }}
          </button>
        </div>
      </div>
    </div>
    
    <div class="input-area">
      <input v-model="inputMessage" 
             @keyup.enter="sendMessage"
             placeholder="输入消息..."
             :disabled="loading" />
      <button @click="sendMessage" :disabled="loading || !inputMessage.trim()">
        {{ loading ? '发送中...' : '发送' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted } from 'vue'

const messages = ref([])
const inputMessage = ref('')
const loading = ref(false)
const messagesContainer = ref(null)
const conversationId = ref('')

const API_KEY = 'app-Orfc1q7yvnIRAo1MIWkOhXzv'
const API_URL = 'https://api.dify.ai/v1/chat-messages'

const addMessage = (content, type, options = null) => {
  messages.value.push({
    id: Date.now(),
    content,
    type,
    showOptions: !!options,
    options
  })
  nextTick(() => {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  })
}

const selectOption = (option) => {
  // 隐藏选项按钮
  messages.value.forEach(msg => {
    if (msg.showOptions) msg.showOptions = false
  })
  // 发送选择的选项
  addMessage(option, 'user')
  sendMessageToAPI(option)
}

onMounted(() => {
  addMessage('你想背哪首诗呢？', 'assistant', ['李白的诗', '春望', '随便'])
})

const sendMessageToAPI = async (message) => {
  loading.value = true
  
  try {
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        inputs: {},
        query: message,
        response_mode: 'blocking',
        conversation_id: conversationId.value,
        user: 'user-123'
      })
    })
    
    const data = await response.json()
    
    if (data.answer) {
      addMessage(data.answer, 'assistant')
      conversationId.value = data.conversation_id
    } else {
      addMessage('抱歉，出现了错误', 'assistant')
    }
  } catch (error) {
    addMessage('网络错误，请重试', 'assistant')
  } finally {
    loading.value = false
  }
}

const sendMessage = async () => {
  if (!inputMessage.value.trim() || loading.value) return
  
  const userMessage = inputMessage.value
  addMessage(userMessage, 'user')
  inputMessage.value = ''
  
  await sendMessageToAPI(userMessage)
}
</script>

<style scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: 600px;
  width: 100%;
  box-sizing: border-box;
  border: 1px solid #ddd;
  border-radius: 8px;
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  background: #f9f9f9;
}

.message {
  margin-bottom: 12px;
  display: flex;
  flex-direction: column;
}

.message.user {
  align-items: flex-end;
}

.message.assistant {
  align-items: flex-start;
}

.content {
  max-width: min(80%, 640px);
  padding: 8px 12px;
  border-radius: 12px;
  word-wrap: break-word;
}

.user .content {
  background: #007bff;
  color: white;
}

.assistant .content {
  background: white;
  border: 1px solid #ddd;
}

.input-area {
  display: flex;
  padding: 16px;
  border-top: 1px solid #ddd;
  gap: 8px;
}

.input-area input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  outline: none;
}

.input-area button {
  padding: 8px 16px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.input-area button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.options {
  margin-top: 8px;
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.option-btn {
  padding: 6px 12px;
  background: #f0f0f0;
  border: 1px solid #ddd;
  border-radius: 16px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.2s;
}

@media (max-width: 480px) {
  .content {
    max-width: 90%;
  }
}
</style>