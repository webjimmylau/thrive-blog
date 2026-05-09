<script setup lang="ts">
import { ref } from "vue";
import io from "socket.io-client";
import { useChatStore } from "@/stores";
import { FormInstance } from "element-plus";
import { getChatDataAPI } from "@/api/Chat";

const store = useChatStore();

// 登记框
const model = ref<boolean>(false);

const form = ref<FormInstance>();

// 用户信息
const chatUserInfo = reactive<ChatUserInfo>({
  name: "",
  avatar: "Ginger",
});

// 头像列表
const avatars = [
  "Ginger",
  "Patches",
  "Sadie",
  "Casper",
  "Molly",
  "Smokey",
  "Lilly",
];
const avatarFilter = (v: string) =>
  `https://api.dicebear.com/7.x/fun-emoji/svg?seed=${v}`;

// 名称校验规则
const rules = reactive({
  name: [
    { required: true, message: "名称不能为空", trigger: "blur" },
    {
      min: 1,
      max: 10,
      message: "名称长度限制为 1 ~ 10个字符",
      trigger: "blur",
    },
  ],
});

// 关闭弹框触发
const close = () => {
  // form.value?.resetFields()
  // chatUserInfo.name = ""
  // chatUserInfo.avatar = "Ginger"
};

// 提交表单触发
const submit = async () => {
  if (!form.value) return;
  await form.value.validate((valid, fields) => {
    if (valid) {
      store.updateChatUserInfo(chatUserInfo);

      model.value = false;

      ElMessage({
        message: "🎉选择成功~",
        type: "success",
      });
    } else {
      console.log("error submit!", fields);
    }
  });
};

// 输入的内容
const content = ref<string>("");

// 记录房间聊天内容
const roomChatList = reactive<{ [room: number]: ChatInfo[] }>({});

// 新增聊天记录时做一些处理
const addChat = (data: ChatInfo) => {
  if (roomChatList[store.room as number]) {
    roomChatList[store.room as number].push(data);
  } else {
    roomChatList[store.room as number] = [];
    roomChatList[store.room as number].push(data);
  }
};

// 从数据库读取聊天记录
const getChatList = async (room: number) => {
  const { data } = await getChatDataAPI(room, { page: 1, size: 5 });

  data.result.forEach((item) => addChat(item.data));
};

// 即时通讯核心代码
const socket = io("http://localhost:9003"); // 替换为你的 Flask-SocketIO 服务器地址

// 选择 | 加入房间
watch(
  () => store.room,
  (room) => {
    socket.emit("joinRoom", room);

    roomChatList[store.room as number] = [];

    getChatList(store.room as number);
  },
  { immediate: true },
);

// 接收该房间的消息
socket.on("roomMsg", (data: ChatInfo) => {
  addChat(data);

  // 发送成功后清空输入框
  content.value = "";
});

// 发送消息
const sendMsg = () => {
  // 没有选择身份，不允许发送消息
  if (store.isNull()) {
    model.value = true;
    return;
  }

  // 不允许发送空消息
  if (!(content.value.trim() && content.value.length)) {
    return ElMessage({
      message: "请输入内容~",
      type: "error",
    });
  }

  // 加入房间
  socket.emit("joinRoom", store.room);

  // 在这个房间中发送消息
  socket.emit("roomMsg", {
    avatar: avatarFilter(store.chatUserInfo?.avatar as string),
    name: store.chatUserInfo?.name as string,
    content: content.value,
    date: "2023-05-25",
  });
};

// 监听Ctrl+Enter组合事件
const handleKeyDown = (e: KeyboardEvent) => {
  if (e.key === "Enter") sendMsg();
};
</script>

<template>
  <div class="Chat">
    <!-- 对话用户信息 -->
    <div class="header">宇阳</div>

    <!-- 聊天框 -->
    <div class="list">
      <div
        v-for="(item, index) in roomChatList[store.room as number]"
        :key="index"
        class="msg"
      >
        <div :class="item.name === store.chatUserInfo?.name ? 'self' : ''">
          <p class="name">{{ item.name }}</p>

          <div class="info">
            <img :src="item.avatar" v-avatar class="avatar" />

            <p class="content">{{ item.content }}</p>
          </div>
        </div>
      </div>
    </div>

    <div class="reply">
      <textarea v-model="content" @keydown="handleKeyDown"></textarea>

      <div class="send" @click="sendMsg">
        <el-button type="primary" plain>发送 Ctrl + Enter</el-button>
      </div>
    </div>

    <el-dialog v-model="model" title="选择一个身份" width="500" @close="close">
      <el-form ref="form" :model="chatUserInfo" :rules="rules">
        <el-form-item label="名称" prop="name">
          <el-input
            v-model="chatUserInfo.name"
            autocomplete="off"
            style="width: 300px"
          />
        </el-form-item>

        <el-form-item label="头像">
          <el-radio-group v-model="chatUserInfo.avatar" class="ml-4">
            <el-radio
              :label="item"
              size="large"
              v-for="(item, index) in avatars"
              :key="index"
            >
              <img :src="avatarFilter(item)" v-avatar alt="" />
            </el-radio>
          </el-radio-group>
        </el-form-item>

        <el-form-item>
          <el-button
            type="primary"
            size="large"
            style="width: 100%"
            @click="submit"
            >选择</el-button
          >
        </el-form-item>
      </el-form>
    </el-dialog>
  </div>
</template>

<style scoped lang="scss">
.Chat {
  flex: 1;
  position: relative;
  background-color: #f4f4f4;

  // 对话信息
  .header {
    height: 80px;
    line-height: 80px;
    padding-left: 40px;
    border-bottom: 1px solid #eee;
    background-color: #fff;
    font-weight: 900;
    font-size: 18px;
  }

  .list {
    overflow: scroll;
    padding: 20px;
    height: 65%;

    .msg {
      margin-bottom: 10px;

      .name {
        font-size: 14px;
        margin-bottom: 5px;
        margin-left: 5px;
        color: #606060;
      }

      .info {
        display: flex;
        align-items: center;
      }

      .self {
        .name {
          text-align: end;
          margin-left: 0;
          margin-right: 5px;
        }

        .info {
          flex-direction: row-reverse;

          .content {
            margin-left: 0;
            margin-right: 10px;
            color: #fff;
            background-color: #539dfd;
          }
        }
      }
    }

    .avatar {
      width: 50px;
      height: 50px;
      border-radius: $round;
    }

    .content {
      background-color: #fff;
      padding: 15px;
      margin-left: 10px;
      @include container;
    }
  }

  // 评论框
  .reply {
    position: absolute;
    bottom: 0;
    width: 100%;
    height: 25%;
    border-top: 1px solid #eee;
    background-color: #fff;

    textarea {
      width: 100%;
      height: 200px;
      padding: 20px;
      border-radius: 5px;
      box-sizing: border-box;
      font-size: 26px;
      font-weight: 900;
      // font-family: "LXGWWenKai";
      transition: all $move;
      outline: none;
      resize: none;
      border: none;
    }

    .send {
      position: absolute;
      bottom: 20px;
      right: 60px;
      width: 100px;
    }
  }

  :deep(.el-form-item__content) {
    .el-radio.el-radio--large {
      margin-bottom: 30px;
      margin-top: 10px;

      img {
        width: 50px;
        height: 50px;
        border-radius: 15px;
      }
    }
  }
}

.el-form {
  .el-form-item {
    &:last-child {
      margin-bottom: 0;
    }
  }
}
</style>
