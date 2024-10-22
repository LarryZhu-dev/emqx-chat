<template>
  <div class='ChatRoomBox'>
    <div class="header">
      <div>
        <span>Emqx-Chat</span>
        <span>Topic: {{ topic }}</span>
        <span>{{ `当前话题在线：${Guard_onlineUsersCount > onlineUsers.length ? Guard_onlineUsersCount : onlineUsers.length}`
          }}</span>
      </div>
      <div>
        <a class="iconfont icon-github" href="https://github.com/LarryZhu-dev/emqx-chat" target="_blank"></a>
        <!-- <span class="iconfont icon-wechat-fill"></span> -->
      </div>
    </div>
    <div class="messages" ref="message">
      <div class="message" v-for="msg in messages" :key="msg.id">
        <div class="messageItemHeader">
          <span class="username" :style="{ color: msg.randomColor }">
            {{ msg.username }}
          </span>
          <span>说：</span>
        </div>
        <div class="messageItemContent">
          <template v-if="isBase64Image(msg.text)">
            <img :src="msg.text" alt="Image" class="msg-img" />
          </template>
          <template v-else>
            {{ msg.text }}
          </template>
        </div>
      </div>
    </div>
    <div class="user-content">
      <div v-if="isBase64Image(inputImg)" class="imagePreview">
        <img :src="inputImg" alt="Image Preview" />
      </div>
      <div class="inputArea">
        <input type="text" v-model="inputText" @keyup.enter="sendMessage" />
        <button @click="sendMessage" style="color: #56fd01">Send</button>
      </div>
    </div>
  </div>
</template>

<script setup lang='ts'>
import mqtt from 'mqtt';
import { ref, onUnmounted, nextTick, onMounted } from 'vue';
import { Buffer } from 'buffer';
import autolog from 'autolog.js';

const messages = ref<{
  id: number;
  text: string;
  clientId: string;
  username: string;
  randomColor: string;
}[]>([]);
const onlineUsers = ref<string[]>([]);

const inputText = ref('');
const inputImg = ref('')
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
type QoS = 0 | 1 | 2
const clientId = generateClientId()
const username = generateClientId()
const password = generateClientId()
// 创建客户端实例
const options = {
  // Clean session
  clean: true,
  connectTimeout: 4000,
  // 认证信息
  clientId: clientId,
  username: username,
  password: password,
  // 配置遗言
  will: {
    topic: `${topic}/will`,
    payload: Buffer.from(clientId),
    qos: 1 as QoS,
    retain: false
  }
}
const client = mqtt.connect(url, options)
client.on('connect', function () {
  console.log('Connected')
  // 订阅主题
  client.subscribe(topic, function (err) {
    if (!err) {
      autolog.log('连接成功', 'success')
      heartbeat()
    }
  })
  client.subscribe(`${topic}/heartbeat`)
  client.subscribe(`${topic}/will`)
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
client.on('message', function (__topic, message) {
  switch (__topic) {
    case `${topic}`:
      handleBaseTopic(message)
      return
    case `${topic}/heartbeat`:
      handleHeartbeat(message)
      return
    case `${topic}/will`:
      handleWill(message)
      return
  }

})
// 处理基本消息
function handleBaseTopic(message: Buffer) {
  let messageObj
  try {
    messageObj = JSON.parse(message.toString())
  } catch (e) {
    messageObj = message.toString()
  }
  if (messages.value.length > 500) {
    // 删除前250条消息
    messages.value.splice(0, 250)
  }
  messages.value.push({ ...messageObj });
  // 滚动到底部
  nextTick(() => {
    let messagesBox = document.querySelector('.messages')!
    const { scrollTop, clientHeight, scrollHeight } = messagesBox;
    let distanceToBottom = scrollHeight - (scrollTop + clientHeight);
    if (distanceToBottom <= 100) {
      messagesBox.scrollTo({
        top: messagesBox.scrollHeight,
        behavior: 'smooth'
      })
    }
  })
}
// 处理心跳
const Guard_onlineUsersCount = ref(0)
function handleHeartbeat(message: Buffer) {
  let clientIdWithOnlineCount = message.toString()
  let [clientId, onlineCount] = clientIdWithOnlineCount.split('_')
  if (onlineCount) {
    Guard_onlineUsersCount.value = +onlineCount
  }
  if (!onlineUsers.value.includes(clientId)) {
    onlineUsers.value.push(clientId)
  }
}
// 处理遗言
function handleWill(message: Buffer) {
  let clientId = message.toString()
  if (onlineUsers.value.includes(clientId)) {
    onlineUsers.value = onlineUsers.value.filter((user) => user !== clientId)
    Guard_onlineUsersCount.value--
  }
}
// 向topic/heartbeat发送心跳包
function heartbeat() {
  client.publish(`${topic}/heartbeat`, `${options.clientId}_${onlineUsers.value.length}`)
  setInterval(() => {
    client.publish(`${topic}/heartbeat`, `${options.clientId}_${onlineUsers.value.length}`)
  }, 15000)
}

// 生成唯一clientId
function generateClientId() {
  return 'user-' + Math.random().toString(16).substr(2, 8)
}
// 发送消息
const randomColor = '#' + Math.floor(Math.random() * 0xffffff).toString(16);
function sendMessage() {
  if (!inputText.value.trim() && !inputImg.value) return
  let message = {
    clientId: options.clientId,
    username: options.username,
    id: generateUUID(),
    text: inputImg.value && isBase64Image(inputImg.value) ? inputImg.value : inputText.value.trim(),
    randomColor: randomColor
  }
  if (message.text === '') return
  let messageBuffer = Buffer.from(JSON.stringify(message))
  client.publish(topic, messageBuffer)
  if (inputText.value) {
    inputText.value = '';
  }
  if (inputImg.value) {
    inputImg.value = '';
  }
}

const isBase64Image = (str: string) => {
  const base64ImageRegex = /^data:image\/(png|jpeg|gif|bmp|webp);base64,([A-Za-z0-9+/]+={0,2})$/;
  return base64ImageRegex.test(str) ? true : false
}

/**
 * @description 图片处理为base64
 */
const getImgFile = async (file: File) => {
  if (currentPasteIsText.value) {
    currentPasteIsText.value = false
    return
  }
  if (file == null) return autolog.log("Please select an image", "error", 1000);
  if (!/(jpg|jpeg|png|GIF|JPG|PNG|gif|webp)$/.test(file.type)) {
    return autolog.log("Only images are allowed！", "error", 1000);
  } else {
    const reader = new FileReader();
    reader.onload = async function (event: any) {
      event.target.result // 获取Base64编码的字符串
      try {
        inputImg.value = event.target.result
      } catch (error) {
        throw new Error('Error')
      }

    };
    // 以DataURL形式读取文件
    reader.readAsDataURL(file);
  }
}

// 销毁客户端实例
onUnmounted(() => {
  client.end()
})

const currentPasteIsText = ref(false)

onMounted(() => {
  window.addEventListener("paste", async (event) => {
    let items = event.clipboardData && event.clipboardData.items;
    let file: any = null;
    if (items && items.length) {
      for (let i = 0; i < items.length; i++) {
        if (items[i].type.includes('text')) {
          currentPasteIsText.value = true
        }
        if (items[i].type.indexOf('image') !== -1) {
          file = items[i].getAsFile();
          break;
        }
      }
    }
    await getImgFile(file)
  });
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

  .header {
    padding: 0 10px 10px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    width: 100%;
    gap: 10px;

    >div {
      display: flex;
      gap: 10px;

      span,
      a {
        cursor: pointer;
        color: aliceblue;

        &.iconfont {
          font-size: 24px;
        }
      }
    }
  }

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
      color: #fff;

      .Image {
        max-width: 50%;
        max-height: 50%;
        object-fit: cover;
      }

      span {
        color: #fff;
      }

      .messageItemHeader {
        display: flex;

        .username {
          color: #42b883;
        }

        .msg-img {
          width: 400px;
          height: 350px;
          object-fit: contain;
          border: 1px solid #d32828;
        }
      }
    }
  }


  .user-content {
    width: 100%;
    margin-top: 12px;

    .inputArea {
      display: flex;
      justify-content: space-between;
      align-items: center;
      width: 100%;

      input {
        flex: 1;
        height: 40px;
        font-size: 16px;
        border-radius: 6px;
        padding: 0 12px;
        margin-right: 12px;
        border: none;
        background-color: #313131;
        color: #fff;
      }

      button {
        background-color: #313131;
      }
    }

    .imagePreview {
      margin-top: 10px;
    }

    .imagePreview img {
      width: 50px;
      height: 50px;
    }
  }

}
</style>
