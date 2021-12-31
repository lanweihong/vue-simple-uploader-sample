<template>
  <div class="container">
    <uploader 
      ref="uploader"
      :options="options"
      :autoStart="false"
      :file-status-text="fileStatusText"
      @file-added="onFileAdded"
      @file-success="onFileSuccess"
      @file-error="onFileError"
      @file-progress="onFileProgress"
      class="uploader-example">
      <uploader-unsupport></uploader-unsupport>
      <uploader-drop>
        <p>拖动文件到这里上传</p>
        <uploader-btn>选择文件</uploader-btn>
      </uploader-drop>
      <uploader-list></uploader-list>
    </uploader>
  </div>
</template>

<script>
import SparkMD5 from 'spark-md5'
const FILE_UPLOAD_ID_KEY = 'file_upload_id'
// 分片大小，10MB
const CHUNK_SIZE = 10 * 1024 * 1024
  export default {
    data () {
      return {
        options: {
          // 上传地址
          target: 'http://127.0.0.1:8086/api/file/upload',
          // 是否开启服务器分片校验。默认为 true
          testChunks: true,
          // 分片大小
          chunkSize: CHUNK_SIZE,
          // 并发上传数，默认为 3 
          simultaneousUploads: 3,
          /**
           * 判断分片是否上传，秒传和断点续传基于此方法
           */
          checkChunkUploadedByResponse: (chunk, message) => {
            let messageObj = JSON.parse(message)
            let dataObj = messageObj.data
            if (dataObj.uploaded !== undefined) {
              return dataObj.uploaded
            }
            // 判断文件或分片是否已上传，已上传返回 true
            // 这里的 uploadedChunks 是后台返回
            return (dataObj.uploadedChunks || []).indexOf(chunk.offset + 1) >= 0
          }
        },
        // 修改上传状态
        fileStatusTextObj: {
          success: '上传成功',
          error: '上传错误',
          uploading: '正在上传',
          paused: '停止上传',
          waiting: '等待中'
        },
        uploadIdInfo: null,
        uploadFileList: [],
        fileChunkList: []
      }
    },
    methods: {
      onFileAdded(file, event) {
        // 有时 fileType为空，需截取字符
        console.log('文件类型：' + file.fileType)
        // 文件大小
        console.log('文件大小：' + file.size + 'B')
        // 1. todo 判断文件类型是否允许上传
        // 2. 计算文件 MD5 并请求后台判断是否已上传，是则取消上传
        console.log('校验MD5')
        this.getFileMD5(file, md5 => {
          if (md5 != '') {
            // 修改文件唯一标识
            file.uniqueIdentifier = md5
            // 请求后台判断是否上传
            // 恢复上传
            file.resume()
          }
        })
      },
      onFileSuccess(rootFile, file, response, chunk) {
        alert('上传成功')
      },
      onFileError(rootFile, file, message, chunk) {
        console.log('上传出错：' + message)
      },
      onFileProgress(rootFile, file, chunk) {
        console.log(`当前进度：${Math.ceil(file._prevProgress * 100)}%`)
      },
      getFileMD5(file, callback) {
        let spark = new SparkMD5.ArrayBuffer()
        let fileReader = new FileReader()
        let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
        let currentChunk = 0
        let chunks = Math.ceil(file.size / CHUNK_SIZE)
        let startTime = new Date().getTime()
        file.pause()
        loadNext()
        fileReader.onload = function(e) {
          spark.append(e.target.result)
          if (currentChunk < chunks) {
            currentChunk++
            loadNext()
          } else {
            let md5 = spark.end()
            console.log(`MD5计算完毕：${md5}，耗时：${new Date().getTime() - startTime} ms.`)
            callback(md5)
          }
        }
        fileReader.onerror = function() {
          this.$message.error('文件读取错误')
          file.cancel()
        }
        function loadNext() {
          const start = currentChunk * CHUNK_SIZE
          const end = ((start + CHUNK_SIZE) >= file.size) ? file.size : start + CHUNK_SIZE
          fileReader.readAsArrayBuffer(blobSlice.call(file.file, start, end))
        }
      },
      fileStatusText(status, response) {
        if (status === 'md5') {
          return '校验MD5'
        } else {
          return this.fileStatusTextObj[status]
        }
      },
      saveFileUploadId(data) {
        localStorage.setItem(FILE_UPLOAD_ID_KEY, data)
      }
    }
  }
</script>

<style>
.uploader-example {
  width: 880px;
  padding: 15px;
  margin: 40px auto 0;
  font-size: 12px;
  box-shadow: 0 0 10px rgba(0, 0, 0, .4);
}
.uploader-example .uploader-btn {
  margin-right: 4px;
}
.uploader-example .uploader-list {
  max-height: 440px;
  overflow: auto;
  overflow-x: hidden;
  overflow-y: auto;
}
</style>