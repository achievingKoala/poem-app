<template>
  <div class="chat-container">
    <div class="header">
      <h1>唐诗助手</h1>
    </div>
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
const userIP = ref('')
const currentQuestion = ref(null)

const API_KEY = 'app-Orfc1q7yvnIRAo1MIWkOhXzv'
const API_URL = 'https://api.dify.ai/v1/chat-messages'

const getUserIP = async () => {
  try {
    const response = await fetch('https://api.ipify.org?format=json')
    const data = await response.json()
    return data.ip
  } catch {
    return 'unknown'
  }
}

const checkDailyLimit = () => {
  const today = new Date().toDateString()
  const key = `usage_${userIP.value}_${today}`
  const count = parseInt(localStorage.getItem(key) || '0')
  return count < 300
}

const incrementUsage = () => {
  const today = new Date().toDateString()
  const key = `usage_${userIP.value}_${today}`
  const count = parseInt(localStorage.getItem(key) || '0')
  localStorage.setItem(key, (count + 1).toString())
}

const getRemainingCount = () => {
  const today = new Date().toDateString()
  const key = `usage_${userIP.value}_${today}`
  const count = parseInt(localStorage.getItem(key) || '0')
  return 300 - count
}

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

onMounted(async () => {
  userIP.value = await getUserIP()
  addMessage('你想背哪首诗呢？', 'assistant', ['李白的诗', '春望', '杜甫的诗', '随便'])
})
// TODO: #1 每五句统计一下正确率
const sendMessageToAPI = async (message) => {
  if (!checkDailyLimit()) {
    addMessage('今日使用次数已达上限（300次），请明天再来', 'assistant')
    return
  }
  
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
        user: userIP.value
      })
    })
    
    const data = await response.json()
    
    if (data.answer) {
      let poemData = null
      try {
        const jsonMatch = data.answer.match(/```json\s*([\s\S]*?)\s*```/)
        if (jsonMatch) {
          poemData = JSON.parse(jsonMatch[1])
        }
      } catch (e) {
        console.log('JSON解析失败:', e)
      }
      
      if (poemData && poemData.title) {
        const isAskPrevious = Math.random() < 0.5
        currentQuestion.value = {
          ...poemData,
          isAskPrevious,
          expectedAnswer: isAskPrevious ? poemData.next_line : poemData.previous_line
        }
        
        const poemMessage = isAskPrevious 
          ? `《${poemData.title}》- ${poemData.author}\n\n上句：${poemData.previous_line}\n下句：？`
          : `《${poemData.title}》- ${poemData.author}\n\n上句：？\n下句：${poemData.next_line}`
        
        incrementUsage()
        addMessage(poemMessage, 'assistant')
        conversationId.value = data.conversation_id
      } else {
        incrementUsage()
        addMessage(data.answer, 'assistant')
        conversationId.value = data.conversation_id
      }
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
  
  if (currentQuestion.value) {
    const isCorrect = userMessage.includes(currentQuestion.value.expectedAnswer)
    const feedback = isCorrect ? '✓ 正确！' : `✗ 错误，正确答案是：${currentQuestion.value.expectedAnswer}`
    addMessage(feedback, 'assistant')
    currentQuestion.value = null
    
    setTimeout(() => {
      sendMessageToAPI('continue')
    }, 1000)
  } else {
    await sendMessageToAPI(userMessage)
  }
}
</script>

<style scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: 90vh;
  width: 100%;
  box-sizing: border-box;
  border: 1px solid #d4af37;
  border-radius: 8px;
  background: linear-gradient(135deg, #faf8f3 0%, #f5f1e8 100%);
}

.header {
  padding: 16px;
  background: linear-gradient(135deg, #2c5530 0%, #1a3d1f 100%);
  color: #d4af37;
  text-align: center;
  border-bottom: 1px solid #d4af37;
}

.header h1 {
  margin: 0;
  font-size: 20px;
  text-shadow: 0 1px 2px rgba(0,0,0,0.3);
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  background: transparent;
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
  background: linear-gradient(135deg, #c8102e 0%, #a00d26 100%);
  color: white;
  box-shadow: 0 2px 4px rgba(200, 16, 46, 0.2);
}

.assistant .content {
  background: rgba(255, 255, 255, 0.9);
  border: 1px solid #d4af37;
  box-shadow: 0 2px 4px rgba(212, 175, 55, 0.1);
}

.input-area {
  display: flex;
  padding: 16px;
  border-top: 1px solid #d4af37;
  gap: 8px;
  background: rgba(255, 255, 255, 0.5);
}

.input-area input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #d4af37;
  border-radius: 4px;
  outline: none;
  background: rgba(255, 255, 255, 0.9);
}

.input-area input:focus {
  border-color: #2c5530;
  box-shadow: 0 0 0 2px rgba(44, 85, 48, 0.1);
}

.input-area button {
  padding: 8px 16px;
  background: linear-gradient(135deg, #2c5530 0%, #1a3d1f 100%);
  color: #d4af37;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 500;
}

.input-area button:hover:not(:disabled) {
  background: linear-gradient(135deg, #1a3d1f 0%, #0f2912 100%);
}

.input-area button:disabled {
  background: #ccc;
  color: #666;
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
  background: rgba(255, 255, 255, 0.8);
  border: 1px solid #d4af37;
  border-radius: 16px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
  color: #2c5530;
}

.option-btn:hover {
  background: #d4af37;
  color: white;
  transform: translateY(-1px);
}

@media (max-width: 480px) {
  .content {
    max-width: 90%;
  }
}
</style>