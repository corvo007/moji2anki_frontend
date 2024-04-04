<template>
  <div>
    <el-row class="input-container">
      <el-col :span="24" class="input-col">
        <span class="input-label">请输入词单URL</span>
        <el-input v-model="url" placeholder="Enter Word List URL" class="input-box"></el-input>
      </el-col>
    </el-row>
    <el-row class="button-container" justify="center">
      <el-col :span="24">
        <el-button type="primary" :loading="creatingApkg" @click="createApkg" :disabled="creatingApkg">Create
          APKG</el-button>
      </el-col>
    </el-row>
    <el-row class="log-row" justify="center">
      <el-col :span="24">
        <el-card class="log-card">
          <div class="log-header">Activity Log</div>
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
    const creatingApkg = ref(false);
    const taskID = ref('');
    const progressLog = ref('');
    const downloadLink = ref('');
    const fileName = ref('');

    const createApkg = () => {
      creatingApkg.value = true;
      progressLog.value = "Creating APKG file...\n";

      axios.get('http://127.0.0.1:8000/create-apkg/', {
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
        });
    };

    const pollLog = () => {
      let intervalId = null;
      if (!intervalId) {
        intervalId = window.setInterval(() => {
          axios.get(`http://127.0.0.1:8000/progress-log/${taskID.value}`)
            .then(response => {
              clearProgressLog(); // 清空进度日志
              clearDownloadLink(); // 清除下载链接信息
              const logString = response.data.progress_log.join('\n');
              progressLog.value += (response.data.progress_log || []).join('\n') + '\n';
              if (logString.includes("SUCCESS")) {
                if (intervalId){ window.clearInterval(intervalId);intervalId=null }
                downloadApkg();
              } else if (logString.includes("Failed")) {
                progressLog.value += "<span style='color: red;'>Failed: Task encountered an error</span>\n";
                if (intervalId) { window.clearInterval(intervalId); intervalId = null }
              }
            })
            .catch(error => {
              progressLog.value += `<span style="color: red;">Error: ${error.response?.data?.detail || "Polling failed"}</span>\n`;
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
      axios.get(`http://127.0.0.1:8000/download-apkg/${taskID.value}`, { responseType: 'blob' })
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
      progressLogCleaned
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
