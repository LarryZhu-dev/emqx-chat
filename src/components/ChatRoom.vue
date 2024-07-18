<template>
  <div class='ChatRoomBox'>
    <div class="messages">
      <div class="message" v-for="msg in messages" :key="msg.id">
        <div class="messageItemHeader">
          <span class="username" :style="{ color: msg.randomColor }">
            {{ msg.username }}
          </span>
          <span>说：</span>
        </div>
        <div class="messageItemContent">
          {{ msg.text }}
        </div>
      </div>
    </div>
    <div class="inputArea">
      <input type="text" v-model="inputText" @keyup.enter="sendMessage" />
      <button @click="sendMessage">Send</button>
    </div>
  </div>
</template>

<script setup lang='ts'>
import mqtt from 'mqtt';
import { ref, onUnmounted, nextTick } from 'vue';
import { Buffer } from 'buffer';
import autolog from 'autolog.js';

const messages = ref<{
  id: number;
  text: string;
  clientId: string;
  username: string;
  randomColor: string;
}[]>([]);
const inputText = ref('');

/***
 * 浏览器环境
 * 使用协议为 ws 和 wss 的 MQTT over WebSocket 连接
 * EMQX 的 ws 连接默认端口为 8083，wss 为 8084
 * 注意需要在连接地址后加上一个 path, 例如 /mqtt
 */
const url = 'wss://broker.emqx.io:8084/mqtt'
/***
 * Node.js 环境
 * 使用协议为 mqtt 和 mqtts 的 MQTT over TCP 连接
 * EMQX 的 mqtt 连接默认端口为 1883，mqtts 为 8084
 */
// const url = 'mqtt://broker.emqx.io:1883'
// 在地址参数中取 topic
let params = new URLSearchParams(window.location.search)
let topic = params.get('topic') as string
if (!topic) {
  topic = '不朽堡垒'
}
// 创建客户端实例
const options = {
  // Clean session
  clean: true,
  connectTimeout: 4000,
  // 认证信息
  clientId: generateClientId(),
  username: generateClientId(),
  password: generateClientId(),
}
const client = mqtt.connect(url, options)
client.on('connect', function () {
  console.log('Connected')
  // 订阅主题

  client.subscribe(topic, function (err) {
    if (!err) {
      autolog.log('连接成功', 'success')
    }
  })
})
// 连接失败
client.on('error', function () {
  autolog.log('连接错误', 'error')
})
// 重新连接
client.on('reconnect', function () {
  autolog.log('重新连接中...', 'info')
})
// 断开连接
client.on('close', function () {
  autolog.log('连接已关闭', 'info')
})
// 接收消息
client.on('message', function (_topic, message) {
  // message is Buffer
  let messageObj
  try {
    messageObj = JSON.parse(message.toString())
  } catch (e) {
  }
  messages.value.push({ ...messageObj });
  // 滚动到底部
  nextTick(() => {
    let messagesBox = document.querySelector('.messages')!
    messagesBox.scrollTo({
      top: messagesBox.scrollHeight,
      behavior: 'smooth'
    })
  })
  console.log('messages.value::: ', messages.value);
})

// 生成唯一clientId
function generateClientId() {
  return 'user-' + Math.random().toString(16).substr(2, 8)
}
// 发送消息
const randomColor = '#' + Math.floor(Math.random() * 0xffffff).toString(16);
function sendMessage() {
  let message = {
    clientId: options.clientId,
    username: options.username,
    id: generateUUID(),
    text: inputText.value,
    randomColor: randomColor
  }
  let messageBuffer = Buffer.from(JSON.stringify(message))
  client.publish(topic, messageBuffer)
  if (inputText.value) {
    inputText.value = '';
  }
}
// 销毁客户端实例
onUnmounted(() => {
  client.end()
})
// 生成UUID
function generateUUID() {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
    var r = (Math.random() * 16) | 0,
      v = c == 'x' ? r : (r & 0x3) | 0x8;
    return v.toString(16);
  });
}
</script>

<style lang='less' scoped>
.ChatRoomBox {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: flex-start;
  padding: 20px;

  .messages {
    flex: 1;
    background: #4b4b4b93;
    width: 100%;
    border-radius: 12px;
    padding: 12px;
    overflow-y: scroll;

    .message {
      display: flex;
      flex-direction: column;
      justify-content: flex-start;
      align-items: flex-start;
      margin-bottom: 12px;
      gap: 6px;
      max-width: 100%;

      .messageItemHeader {
        display: flex;

        .username {
          color: #42b883;
        }
      }
    }
  }

  .inputArea {
    display: flex;
    justify-content: space-between;
    align-items: center;
    width: 100%;
    margin-top: 12px;

    input {
      flex: 1;
      height: 40px;
      border-radius: 6px;
      padding: 0 12px;
      margin-right: 12px;
      border: none;
    }
  }
}
</style>