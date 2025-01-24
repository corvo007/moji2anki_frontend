<template>
  <div>
    <el-row class="title-row" justify="center">
      <el-col :span="24">
        <h1>APKG 文件生成器</h1>
      </el-col>
    </el-row>
    <el-row class="input-container">
      <el-col :span="24" class="input-col">
        <span class="input-label">请输入词单 URL</span>
        <el-input v-model="url" placeholder="请输入词单 URL" class="input-box"></el-input>
      </el-col>
    </el-row>
    <el-row class="button-container" justify="center">
      <el-col :span="24">
        <el-button type="primary" @click="createApkg" :disabled="processing">生成 Anki 牌组</el-button>
      </el-col>
    </el-row>
    <el-row class="log-row" justify="center">
      <el-col :span="24">
        <el-card class="log-card">
          <div class="log-header">活动日志</div>
          <el-scrollbar wrap class="log-text">
            <pre v-html="progressLogCleaned"></pre>
          </el-scrollbar>
          <div v-if="downloadLink" style="margin-top: 10px;">
            <a :href="downloadLink" download>{{ fileName }}</a>
          </div>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import { defineComponent, ref, computed } from 'vue';
import axios from 'axios';

export default defineComponent({
  setup() {
    const url = ref('');
    const apiHost = ref('');  // API主机地址作为响应式引用
    const creatingApkg = ref(false);
    const taskID = ref('');
    const progressLog = ref('');
    const downloadLink = ref('');
    const fileName = ref('');
    const processing = ref(false); // 定义处理状态，防止重复提交

    const createApkg = () => {

      if (processing.value) {
        console.log("Processing is already true.");
        return; // 如果正在处理，则直接返回，防止重复处理
      }

      creatingApkg.value = true;
      processing.value = true;
      console.log("Processing started.");
      progressLog.value = "Creating APKG file...\n";

      axios.get(`${apiHost.value}/create-apkg/`, {
        params: { url: url.value }
      })
        .then(response => {
          taskID.value = response.data.task_id;
          pollLog();
          creatingApkg.value = false;
        })
        .catch(error => {
          clearProgressLog(); // 清空进度日志
          clearDownloadLink(); // 清除下载链接信息
          progressLog.value += `<span style="color: red;">Error: ${error.response?.data?.detail || "Request failed"}</span>\n`;
          creatingApkg.value = false;
          processing.value = false;
        });
    };

    const pollLog = () => {
      let intervalId = null;
      if (!intervalId) {
        intervalId = window.setInterval(() => {
          axios.get(`${apiHost.value}/progress-log/${taskID.value}`)
            .then(response => {
              clearProgressLog(); // 清空进度日志
              clearDownloadLink(); // 清除下载链接信息
              const logString = response.data.progress_log.join('\n');
              progressLog.value += (response.data.progress_log || []).join('\n') + '\n';
              if (logString.includes("SUCCESS")) {
                if (intervalId){ window.clearInterval(intervalId);intervalId=null }
                progressLog.value += "File downloading...\n";
                downloadApkg();
                processing.value = false;
              } else if (logString.includes("Failed")) {
                progressLog.value += "<span style='color: red;'>Failed: Task encountered an error</span>\n";
                if (intervalId) { window.clearInterval(intervalId); intervalId = null }
                processing.value = false;
              }
            })
            .catch(error => {
              progressLog.value += `<span style="color: red;">Error: ${error.response?.data?.detail || "Polling failed"}</span>\n`;
              processing.value = false;
            });
        }, 1500);
}
      
    };

    const clearProgressLog = () => {
      progressLog.value = '';
    };

    const clearDownloadLink = () => {
      downloadLink.value = '';
      fileName.value = '';
    };

    const downloadApkg = () => {
      axios.get(`${apiHost.value}/download-apkg/${taskID.value}`, { responseType: 'blob' })
        .then(response => {
          const url = window.URL.createObjectURL(new Blob([response.data]));
          const link = document.createElement('a');
          link.href = url;
          link.setAttribute('download', `${taskID.value}.apkg`);
          document.body.appendChild(link);
          link.click();
          progressLog.value += "File downloaded successfully.\n";
          downloadLink.value = url;
          fileName.value = `${taskID.value}.apkg`;
        })
        .catch(error => {
          progressLog.value += `<span style="color: red;">Error: ${error.response?.data?.detail || "Download failed"}</span>\n`;
        });
    };

    const progressLogCleaned = computed(() => progressLog.value.replace(/\\x1b\[31m/g, "<span style='color: red;'>").replace(/\\x1b\[0m/g, "</span>"));

    return {
      url,
      creatingApkg,
      taskID,
      progressLog,
      downloadLink,
      fileName,
      createApkg,
      progressLogCleaned,
      processing
    };
  },
});

</script>

<style scoped>
.input-container {
  margin-top: 10px;
}

.input-col {
  display: flex;
  justify-content: center;
  align-items: flex-start;
}

.input-label {
  align-self: flex-start;
  padding-right: 10px;
}

.input-box {
  max-width: 300px;
}

.button-container {
  margin-top: 10px;
}

.log-row {
  margin-top: 20px;
}

.log-card {
  max-width: 600px;
  margin: 0 auto;
}

.log-header {
  padding: 12px;
}

.log-text {
  overflow-y: auto;
  max-height: 400px;
}
</style>
