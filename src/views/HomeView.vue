<template>
  <el-container>
    <el-header style="height: 100px">
      <div class="header">
        低光照彩色图像增强算法展示
      </div>
    </el-header>
    <el-container>
      <el-aside class="side">
        <el-card class="process-card">
          <template #header>
            <div class="clearfix" style="text-align:center;">
              <span class="steps" style="letter-spacing: 6px;font-weight: bold; font-size: 1.2em;">算法演示步骤</span>
            </div>
          </template>
          <el-steps :active="active" direction="vertical" class="steps">
            <el-step title="步骤1" :icon="Edit">
              <template #description>
                <p style="font-size: 15px">下载示例文件</p>
                <div class="button">
                  <!-- 下载文件 -->
                  <el-button
                    type="primary"
                    @click="download"
                  >下载
                    <template #icon>
                      <el-icon>
                        <Download />
                      </el-icon>
                    </template>
                  </el-button>
                </div>
                <br>
                <br>
              </template>
            </el-step>
            <el-step title="步骤2" :icon="Upload">
              <template #description>
                <p style="font-size: 15px">上传图像至后台服务进行算法增强</p>
                <div class="button">
                  <el-upload
                    :action="urls.baseUrl + urls.upload"
                    :on-success="handleSuccess"
                    :on-error="handleError"
                    :before-upload="beforeUpload"
                    :show-file-list="false"
                  >
                    <el-button
                      type="primary"
                      @click="uploadButton"
                    >上传
                      <template #icon>
                        <el-icon>
                          <UploadFilled />
                        </el-icon>
                      </template>
                    </el-button>
                  </el-upload>
                </div>
                <br>
              </template>
            </el-step>
            <el-step title="步骤3" :icon="Picture">
              <template #description>
                <div>
                  <p style="font-size: 15px">显示算法增强后的图像</p>
                  <br>
                  <br>
                </div>
              </template>
            </el-step>
          </el-steps>
        </el-card>
      </el-aside>
      <el-main style="padding-left: 10px;margin-top: -10px">
        <div>
          <el-card
            :style="'width:'+(imgWidth*2+100)+'px;height: '+(imgHeight+105+100)+'px;'"
          >
            <template #header>
              <div class="feature">
                <div>
                  <span style="letter-spacing: 3px;font-weight: bold; font-size: 1.1em;">图片增强前后对比</span>
                </div>
                <div>
                  <el-upload
                    :action="urls.baseUrl + urls.upload"
                    :on-success="handleSuccess"
                    :on-error="handleError"
                    :before-upload="beforeUpload"
                    :show-file-list="false"
                  >
                    <el-button
                      type="primary"
                      @click="uploadButton"
                    >重新选择图像
                      <template #icon>
                        <el-icon>
                          <UploadFilled />
                        </el-icon>
                      </template>
                    </el-button>
                  </el-upload>
                </div>
              </div>

            </template>
            <div class="image-container">
              <div
                v-loading="rawLoading"
                element-loading-text="正在上传原始数据...">
                <el-image
                  :src="rawImageUrl"
                  :style="'width:'+imgWidth+'px;height: '+imgHeight+'px;'"
                >
                  <template #placeholder>
                    <div class="image-slot">Loading<span class="dot">...</span></div>
                  </template>
                </el-image>
                <!-- 原图文字 -->
                <div class="image" :style="'border-radius:0 0 5px 5px; width='+imgWidth+'px'">
                  <span style="color:white;letter-spacing:6px;">原图像</span>
                </div>
              </div>

              <div style="margin-left: 30px;"
              >
                <div element-loading-text="正在获取预测结果..."
                     v-loading="predictLoading">
                  <el-image
                    :src="predictImageUrl"
                    :style="'width:'+imgWidth+'px;height: '+imgHeight+'px;'"
                  >
                  </el-image>
                  <div class="image" style="border-radius: 0 0 5px 5px;">
                    <span style="color:white;letter-spacing:4px;">算法增强后的图像</span>
                  </div>
                </div>
              </div>
            </div>
          </el-card>
        </div>

      </el-main>
    </el-container>
  </el-container>
</template>

<script setup lang="ts">

import { Edit, Picture, Upload } from '@element-plus/icons-vue'
import { ElMessage, type UploadFile, type UploadFiles, type UploadRawFile } from 'element-plus'
import { ref } from 'vue'
import axios from 'axios'

const imgHeight = ref(360)
const imgWidth = ref(540)

const rawLoading = ref(false)
const predictLoading = ref(false)

const urls = {
  baseUrl: 'http://127.0.0.1:5000',
  upload: '/upload',
  getRaw: '/getRaw',
  getPredict: '/getPredict',
  predict: '/predict',
  demo: '/download_image'
}

const active = ref(0)

const rawImageUrl = ref<string>()
const predictImageUrl = ref<string>()

const handleSuccess = async (response: any, uploadFile: UploadFile, uploadFiles: UploadFiles) => {
// 上传成功的回调
  console.log('File uploaded successfully', response)
  ElMessage.success('图片上传成功')

  rawImageUrl.value = urls.baseUrl + urls.getRaw + '/' + response.filepath.split('/')[1]
  rawLoading.value = false
  active.value = 2
  predictLoading.value = true
  await predict(response.filepath)
}

const handleError = (error: Error, uploadFile: UploadFile, uploadFiles: UploadFiles) => {
// 上传失败的回调
  console.error('File upload failed', error)
}

const beforeUpload = (rawFile: UploadRawFile) => {
  // 在上传前的钩子，可用于限制上传文件的类型和大小等
  const isImage = rawFile.type.startsWith('image/')
  if (!isImage) {
    ElMessage.error('只能上传图片文件')
    return false
  }

  const isLt2M = rawFile.size / 1024 / 1024 < 2
  if (!isLt2M) {
    ElMessage.error('图片大小不能超过 2MB')
    return false
  }

  rawLoading.value = true

  return true // 允许上传
}

const uploadButton = () => {
  active.value = 1
}

const predict = async (filePath: string) => {
  // 发送 GET 请求到 Flask 后端
  const response = await axios.get(urls.baseUrl + urls.predict, {
    params: {
      file_path: filePath
    }
  })
  if (response?.status === 200) {
    predictImageUrl.value = urls.baseUrl + urls.getPredict + '/' + response.data.predict_image_name
    console.log(predictImageUrl.value)
    predictLoading.value = false
    // 预测成功
    return true
  }
  return false
}

const download = () => {
  const apiUrl = urls.baseUrl + urls.demo
  active.value = 1

  // 使用 Axios 发起 GET 请求
  axios({
    method: 'get',
    url: apiUrl,
    responseType: 'blob'  // 指定响应类型为二进制数据
  })
    .then(response => {
      // 创建一个 <a> 元素，模拟点击下载
      const link = document.createElement('a')
      link.href = window.URL.createObjectURL(new Blob([response.data]))
      link.download = 'demo.png'  // 替换为你想要的文件名
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
    })
    .catch(error => {
      console.error('Error downloading image:', error)
    })
}

</script>

<style scoped>
.image-container {
  display: flex;
  justify-content: space-between;
  padding: 10px;
}

.image {
  height: 30px;
  text-align: center;
  background-color: #21b3b9;
  line-height: 30px;
}

.side {
  width: 250px;
  margin-left: 20px;
  margin-top: 10px;
}

:deep(.el-card ) {
  border-radius: 10px;
}

.process-card {
  width: 230px;
}

.steps {
  padding: 0;
}

.header {
  color: #21b3b9;
  font-weight: bold;
  font-size: 2em;
  margin-left: 20px;
  margin-top: 20px;
  letter-spacing: 6px;
}

.file {
  width: 200px;
  height: 130px;
  position: absolute;
  left: -20px;
  top: 0;
  z-index: 1;
  -moz-opacity: 0;
  -ms-opacity: 0;
  -webkit-opacity: 0;
  opacity: 0; /*css属性&mdash;&mdash;opcity不透明度，取值0-1*/
  filter: alpha(opacity=0);
  cursor: pointer;
}

.button {
  margin-left: 10px;
}

.image-feature {
  width: 1000px;
  margin-top: 50px;
}

.feature {
  display: flex;
  justify-content: space-between;
}

:deep(.el-image__inner) {
  border-radius: 3px 3px 0 0;
}
</style>
