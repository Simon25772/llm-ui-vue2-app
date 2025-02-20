<template>
  <div style="margin: 0;
    height: 100vh;
    background-color: #eee;" class="flex">
      <!-- 侧边栏 -->
      <div style="width: 15vw;" class="baiSe">
          <!-- 工具栏 -->
          <div style="padding: 10px;
          align-items: center;
          justify-content: center;
          border-bottom:1px solid #999;" class="flex">
            <!-- 新建按钮 -->
            <el-button type="primary" icon="el-icon-edit" circle></el-button>
          </div>
          <div class="huiHua">会话</div>
          <div class="huiHua">会话</div>
          <div class="huiHua">会话</div>
      </div>
      <!-- 主区域 -->
      <div class="flex flex1 column">
          <!-- 头部栏 -->
          <div style="height: 10vh;" class="baiSe"></div>
          <!-- 内容区 -->
          <div class="flex1" style="white-space: pre-wrap;">{{chatContent}}</div>
          <!--底部栏-->
          <div style="height: 10vh; justify-self: end;justify-content: center;" class="baiSe flex row">
            <!-- 输入框 -->
            <div style="width: 700px;">
              <el-input
                type="textarea"
                placeholder="给GPT发送消息"
                v-model="textarea"
                row="2">
              </el-input>
              <span>内容由 AI 生成，请仔细甄别</span>
            </div>
            <!-- 提交表单按钮 -->
            <div style="justify-content: center;" class="flex column">
              <el-button @click="sendMessage" type="primary" icon="el-icon-top" circle></el-button>
            </div>
          </div>
      </div>
  </div>
</template>

<script>


export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  data () {
    return {
      count: 10,
      loading: false,
      textarea:"",
      chatContent:""
    }
  },
  methods:{
    sendMessage(){
      if(this.textarea==""){
        console.log("消息为空！")
      }else{
        this.chatContent += "\nYou:" + this.textarea + "\nAI:"
        this.fetchStream();
        console.log("send message!")
        this.textarea = ""
      }
    },
    async fetchStream() {
      // 处理流式响应
      try {
        const response = await fetch('http://localhost:8081/chat/'+this.textarea,{
          method:"GET",
          headers:{
            'Content-Type':'application/json',
          },
        });
        const reader = response.body.getReader();
        const decoder = new TextDecoder();
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        console.log("Response headers:", response.headers);

        // 使用 ReadableStream API 处理分块数据
        
        // eslint-disable-next-line no-constant-condition
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;

          // 解码并拼接分块数据
          this.chatContent += decoder.decode(value, { stream: true });
        }

      } catch (error) {
        console.error("Error during fetch or stream processing:", error);
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
*{
    border: 1px solid black;
}
.flex{
    display: flex;
}
.baiSe{
    background-color: white;
}
.flex1{
    flex:1;
}
.column{
    flex-direction: column;
}
.row{
    flex-direction: row;
}
.huiHua{
    padding: 10px 20px;
    border-bottom:1px solid;
}
</style>
