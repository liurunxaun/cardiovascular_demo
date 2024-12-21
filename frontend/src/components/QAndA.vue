<template>
  <div class="container">
    <!-- 左侧：聊天界面 -->
    <div class="chat-container">
      <div class="chat-header">
        <h2>心血管疾病智能问答体</h2>
      </div>
      <div class="chat-body">
        <div v-for="(message, index) in messages" :key="index" class="message">
          <div :class="message.sender === 'user' ? 'user-message' : 'bot-message'" v-html="message.text"></div>
        </div>
      </div>
      <div class="chat-footer">
        <input v-model="userInput" @keyup.enter="sendMessage" placeholder="请输入您的问题..." />
        <button @click="sendMessage">Send</button>
      </div>
    </div>

    <!-- 右侧：知识图谱展示 -->
<!--    <div class="knowledge-graph">-->
<!--      <h2>Knowledge Graph</h2>-->
<!--      <div ref="chart" id="graph"></div> &lt;!&ndash; 用于渲染知识图谱的区域 &ndash;&gt;-->
<!--    </div>-->
  </div>
</template>

<script>
import * as echarts from 'echarts';
import { marked } from 'marked';
import axios from "axios";

export default {
  data() {
    return {
      messages: [],
      userInput: '',
      question: '',
      answers: [],
      answer1: '',
      answer2: [],
      answer3: {},
      isLoading: false
    };
  },
  methods: {
    async sendMessage() {
      if (this.userInput.trim()) {
        this.question = this.userInput;
        this.userInput = '';
        // 用户消息
        this.messages.push({ sender: 'user', text: this.question });

        // 添加“正在分析...”的消息
        this.messages.push({ sender: 'bot', text: '正在分析...' });

        this.isLoading = true;

        try {
          const response = await axios.get("http://10.43.126.232:8080/api/qa", {
            params: { question : this.question }
          });

          this.answers = response.data;
          console.log(this.answers)

          this.answer1 = this.answers[0]
          this.answer2 = this.answers[1]
          this.answer3 = this.answers[2]
          this.answer4 = this.answers[3]

          // 模拟AI的回复
          this.temp_answer = ""

          this.isLoading = false;
          this.messages.pop(); // 删除“正在分析...”消息

          if (this.answer1 != null) {
            this.temp_answer += marked(this.answer1)
          }

          if (this.answer2.length != 0) {
            this.temp_answer += "<b>推理步骤:</b><br>"
            this.temp_answer += this.answer2
          }

          if (this.answer3.length != 0) {
            this.temp_answer += "<br><br><b>知识图谱中检索到的相关信息:</b><br>"
            this.temp_answer += this.answer3
          }

          setTimeout(() => {
            this.messages.push({ sender: 'bot', text: this.temp_answer });
          }, 1000);

          if (this.answer4.length != 0) {
            console.log('Actual Graph Data:', this.answer3);
            // 渲染知识图谱
            this.renderGraph(this.answer3);
          }

        }catch (error) {
          console.error("Error fetching answer:", error);
          this.messages.pop(); // 删除“正在分析...”消息
          // 模拟AI的回复
          setTimeout(() => {
            this.messages.push({ sender: 'bot', text: error});
          }, 1000);
          this.isLoading = false;
        }
      }
    },


    renderGraph(graphs) {
      const graphData = graphs[0]; // 假设只可视化第一个子图
      console.log(graphData)

      const graphContainer = this.$refs.chart;

      // 检查 graphContainer 是否有效
      if (!graphContainer) {
        console.error('Graph container is not found');
        return;
      }

      // 确保 graphData 有效
      if (!graphData || !graphData.nodes || !graphData.edges) {
        console.error('Invalid graph data:', graphData);
        return;
      }

      const seenIds = new Set();
      const uniqueNodes = [];

      // 过滤重复节点
      graphData.nodes.forEach(node => {
        if (!seenIds.has(node.id)) {
          seenIds.add(node.id);
          if (node.type == '疾病') {
            uniqueNodes.push({
              id: node.id,
              name: node.label,
              value: node.type,
              draggable: true,
              label: {
                show: true,
                color: '#2ecc71',  // 设置字体颜色
                fontSize: 16,      // 设置字体大小
                fontFamily: 'Arial'  // 设置字体类型
              }
            });
          } else {
            uniqueNodes.push({
              id: node.id,
              name: node.type,
              value: node.label,
              draggable: true
            });
          }
        }
      });

      const edges = graphData.edges.map(edge => ({
        source: edge.source,
        target: edge.target,
        label: {
          show: false,
          formatter: edge.relationship
        }
      }));

      const option = {
        tooltip: {},
        animation: false,
        series: [
          {
            type: 'graph',
            layout: 'force',
            force: {
              repulsion: 500
            },
            data: uniqueNodes,
            links: edges,
            roam: true,
            label: {
              show: true
            }
          }
        ]
      };

      const chart = echarts.init(graphContainer);
      chart.setOption(option);
    }


  }
};
</script>

<style>
/* 全局样式重置 */
.container {
  display: flex;
  height: 100vh; /* 确保 container 占据整个视口高度 */
}

.chat-container {
  width: 100%;
  border-right: 1px solid #ccc;
  display: flex;
  flex-direction: column;
}

.knowledge-graph {
  width: 40%;
  padding: 10px;
  min-height: 80vh; /* 确保有足够的空间 */
}


.chat-header {
  background-color: #007bff;
  color: white;
  padding: 10px;
  text-align: center;
}
.chat-body {
  flex: 1;
  padding: 10px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}
.message {
  margin-bottom: 10px;
  display: flex; /* 添加这一行 */
}
.user-message {
  text-align: left;
  background: #e0f7fa; /* 用户消息背景色 */
  border-radius: 10px; /* 圆角 */
  padding: 10px; /* 内边距 */
  padding-left: 20px; /* 内边距 */
  margin-left: auto; /* 自动左边距使消息靠右 */
  max-width: 75%; /* 限制最大宽度 */
}

.bot-message {
  text-align: left;
  background: #f1f8e9; /* AI消息背景色 */
  border-radius: 10px; /* 圆角 */
  padding: 10px; /* 内边距 */
  margin-right: auto; /* 自动右边距使消息靠左 */
  max-width: 75%; /* 限制最大宽度 */
}

.answer2-message {
  line-height: 1.8; /* 设置answer2的行间距 */
}

.chat-footer {
  display: flex;
  padding: 10px;
}
.chat-footer input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
}
.chat-footer button {
  padding: 10px;
  margin-left: 10px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
#graph {
  width: 100%;
  height: 80vh;
}
</style>
