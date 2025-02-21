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
            <el-button type="primary" icon="el-icon-edit" circle @click="createSession"></el-button>
          </div>
          <div class="huiHua" v-for="(item,index) in sessions"  :key="index" @click="checkoutSession(index)">{{ index }}:{{ item.title }}</div>
      </div>
      <!-- 主区域 -->
      <div class="flex flex1 column" >
          <!-- 头部栏 -->
          <div style="height: 7vh;" class="baiSe"></div>
          <!-- 内容区 -->
          <!-- <div class="flex1" style="white-space: pre-wrap;">{{chatContent}}</div> -->
          <div class="flex flex1" style="white-space: pre-wrap;overflow: auto;">
            <div v-if="sessions.length>0" class="flex1">
              <div class="xiaoXi flex row" v-for="(item,index) in sessions[currentSessionIndex].history" :key="index">
                <div class="flex1">
                  {{ item.role }}:{{ item.content }}
                </div>
                <div>回滚</div>
              </div>
            </div>
          </div>
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
              <el-button v-if="textarea.length>0&&!isGenerating" @click="sendMessage" type="primary" icon="el-icon-top" circle></el-button>
              <el-button v-else icon="el-icon-top" circle disabled></el-button>
            </div>
          </div>
      </div>
  </div>
</template>

<script>
import axios from "axios";

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
      chatContent:"",
      sessions:[],
      currentSessionIndex:0,
      isGenerating:false,
    }
  },
  methods:{
    checkoutSession(index){
      this.currentSessionIndex=index;
    },
    // 创建新会话
    createSession(){
      axios
      .get("http://localhost:8081/createSession")
      .then((response) => {
        this.sessions = response.data;
      })
      .catch((error) => {
        console.error("Error fetching sessions:", error);
      });
    },
    // 发送消息
    sendMessage(){
      if(this.textarea==""){
        console.log("消息为空！")
      }else{
        this.chatContent += "\nYou:" + this.textarea + "\nAI:"
        this.sessions[this.currentSessionIndex].history.push({role:"User", content:this.textarea});
        this.sessions[this.currentSessionIndex].history.push({role:"AI",content:""});
        this.fetchStream();
        this.textarea = "";
      }
    },
    
    async fetchStream() {
      this.isGenerating=true;
      const id=this.sessions[this.currentSessionIndex].id;
      const content=this.textarea;
      const myIndex=this.currentSessionIndex;
      // 处理流式响应
      try {
        const response = await fetch(`http://localhost:8081/chat?id=${id}&content=${encodeURIComponent(content)}`, {
            method: "GET",
            headers: {
                'Content-Type': 'application/json',
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
          this.sessions[myIndex].history.at(-1).content += decoder.decode(value, { stream: true });
          this.chatContent += decoder.decode(value, { stream: true });
        }
        this.isGenerating=false;
      } catch (error) {
        console.error("Error during fetch or stream processing:", error);
      }
    }
  },
  created() {
    axios
      .get("http://localhost:8081/getAllSessions")
      .then((response) => {
        this.sessions = response.data;
        console.log(response.data)
      })
      .catch((error) => {
        console.error("Error fetching sessions:", error);
      });
  },
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
.xiaoXi{
    padding: 10px 20px;
    border-bottom:1px solid;
    text-align: left;
}
</style>
