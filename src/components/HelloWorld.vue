<template>
  <div style="margin: 0;
    height: 100vh;
    background-color: #eee;" class="flex">
      <!-- 侧边栏 -->
      <div style="width: 15vw;background-color: rgb(247,248,252);font-family: Roboto;" class="felx">
          <!-- 工具栏 -->
          <div style="padding: 10px;
          align-items: center;
          justify-content: center;
          border-bottom:1px solid #999;
          height: 5vh;" class="flex">
            <!-- 新建按钮 -->
            <el-button icon="el-icon-plus" @click="createSession" style="width: 12vw;">新建对话</el-button>
          </div>
            <div class="huiHua flex row" v-for="(item,index) in sessions"  :key="index" @click="checkoutSession(index)">
              <div class="flex1" style="text-align: left;">
                {{ index }}:{{ item.title }}
              </div>
              <div>
                <i class="el-icon-delete"></i>
              </div>
            </div>
      </div>
      <!-- 主区域 -->
      <div class="flex flex1 column baiSe" >
          <!-- 头部栏 -->
          <div style="height: 7vh;justify-content: space-between;" class="baiSe flex">
            <div style="margin: 20px;font-size: 20px;font-family: Fantasy;">Advanced-LM</div>
            <i class="el-icon-user-solid" style="font-size:30px;margin: 20px;"></i>
          </div>
          <!-- 内容区 -->
          <div class="flex flex1" style="white-space: pre-wrap;overflow: auto;font-family: Inter;">
            <div v-if="sessions.length>0" class="flex1">
              <div class="xiaoXi flex row" v-for="(item,index) in sessions[currentSessionIndex].history" :key="index">
                <div v-if="item.role=='User'" class="flex row">
                  <el-image 
                    style="width: 40px;height:40px;border-radius: 50%;"
                    :src="userUrl" 
                    :preview-src-list="srcList">
                  </el-image>
                  <div class="flex1" style="width: 77vw;">
                  {{ item.role }}:<br>{{ item.content }}
                  </div>
                  <i class="el-icon-edit" @click="rollback(index)"></i>
                </div>
                <div v-else class="flex row">
                  <el-image 
                    style="width: 40px;height:40px;border-radius: 50%;"
                    :src="aiUrl" 
                    :preview-src-list="srcList">
                  </el-image>
                  <div class="flex1" style="width: 77vw;">
                  {{ item.role }}:<br>{{ item.content }}
                  </div>
                    <i class="el-icon-refresh-right"></i>
                </div>
              </div>
            </div>
          </div>
          <!--底部栏-->
          <div style="height: 10vh; justify-self: end;justify-content: center;" class="baiSe flex row">
            <!-- 输入框 -->
            <div style="width: 65vw;">
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
              <el-button v-if="sessions.length>currentSessionIndex&&textarea.length>0&&!sessions[currentSessionIndex].isGenerating" @click="sendMessage" type="primary" icon="el-icon-top" circle></el-button>
              <el-button v-else-if="sessions.length>currentSessionIndex&&sessions[currentSessionIndex].isGenerating" icon="el-icon-loading" @click="stopGenerate" circle type="primary"></el-button>
              <el-button v-else icon="el-icon-top" circle :disabled="true"></el-button>
            </div>
          </div>
      </div>
  </div>
</template>

<script>
import axios from "axios";
import Vue from 'vue';

export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  data () {
    return {
      aiUrl: 'https://chat.qwenlm.ai/static/favicon.png',
      userUrl: 'https://cdn.qwenlm.ai/output/64b596f9-e26b-4ba2-a3c0-681a6f1d3df9/t2i/a49fed19-13a9-4dae-8d2e-7492e3ae5a5e/66fbbb3d-d8c1-47d7-85bd-2347da84549b.png',
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
    rollback(index){
      const id=this.sessions[this.currentSessionIndex].id;
      this.textarea=this.sessions[this.currentSessionIndex].history[index].content;
      axios
      .get("http://localhost:8081/rollback", {
        params:{
          id:id,
          index:index
        }
      })
      .then((response) => {
        console.log("rollback");
        this.sessions = response.data;
      })
    },
    stopGenerate(){
      const index=this.currentSessionIndex;
      if (this.sessions[index].controller) {
        this.sessions[index].controller.abort(); // 中断请求
        this.sessions[index].controller = null; // 清理控制器
        Vue.set(this.sessions, index, { 
          ...this.sessions[index], 
          isGenerating: false 
        });
      }else{
        console.log("中断控制器为空!");
      }
    },
    checkoutSession(index){
      this.currentSessionIndex=index;
    },
    // 创建新会话
    createSession(){
      console.log("A")
      axios
      .get("http://localhost:8081/createSession")
      .then((response) => {
        console.log("B")
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
      const id=this.sessions[this.currentSessionIndex].id;
      const content=this.textarea;
      const myIndex=this.currentSessionIndex;
      this.sessions[myIndex].isGenerating=true;
      this.sessions[myIndex].controller=new AbortController();
      const signal = this.sessions[myIndex].controller.signal;
      // 处理流式响应
      try {
        const response = await fetch(`http://localhost:8081/chat?id=${id}&content=${encodeURIComponent(content)}`, {
            method: "GET",
            headers: {
                'Content-Type': 'application/json',
            },
            signal,
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
        }
        this.sessions[myIndex].isGenerating=false;
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
/* *{
  border: solid 1px black;
} */
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
    font-size: 14px;
    /* border-bottom:1px solid; */
}
.xiaoXi{
    padding: 10px 20px;
    /* border-bottom:1px solid; */
    text-align: left;
}
</style>
