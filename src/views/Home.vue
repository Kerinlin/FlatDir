<template>
  <div class="home" @drop.prevent="addFile" @dragover.prevent>
    <div class="input-wrapper">
      <label for="attach-project-file">
        <div class="filename" v-if="!path" id="attached-project-file">
          点击选择或者拖拽文件或者目录
        </div>
        <div class="filename" v-if="path" id="attached-project-file">
          {{ path }}
        </div>
      </label>
      <input
        @change="getFile"
        id="attach-project-file"
        type="file"
        webkitdirectory
        directory
      />
    </div>

    <button class="button loading-btn" v-if="loading">
      <span class="spinner"></span>
      <span class="spinner-text">迁移中...</span>
    </button>
    <button class="button" v-else @click="startTrans(path)">开始迁移</button>
  </div>
</template>

<script>
const path = window.require("path");
const fs = window.require("fs");
const fsp = fs.promises;
const { shell, dialog } = window.require("electron");

export default {
  name: "home",
  data() {
    return {
      path: "",
      loading: false,
      back: false,
      droppedFiles: [],
      isDropMulti: false
    };
  },
  methods: {
    addFile(e) {
      // 将伪数组转换成数组
      this.droppedFiles = [...e.dataTransfer.files];

      // 处理多个文件一起拖拽的情况
      if (this.droppedFiles.length > 1) {
        // 只有在同一个目录下才能多选，所以获取到第一个的父级目录就可以了
        this.path = path.dirname(this.droppedFiles[0].path);
        // 标记是否是多选
        this.isDropMulti = true;
      } else {
        this.path = this.droppedFiles[0].path;
      }
    },
    getFile(e) {
      this.path = e.target.files[0].path;
    },

    // 判断是否是文件夹
    async isDir(path) {
      try {
        let res = await fsp.stat(path);
        if (res.isDirectory()) {
          return true;
        } else {
          return false;
        }
      } catch (error) {
        console.log(error);
      }
    },

    // 获取文件路径下的所有文件
    async getAllFiles(path) {
      try {
        let res = await fsp.readdir(path);
        return res;
      } catch (error) {
        console.log(error);
      }
    },

    /*
    dirPath: 原文件夹
    targetPath: 新的目标文件夹
    */
    async moveFiles(dirPath, targetDirPath) {
      const files = await this.getAllFiles(dirPath);
      files.forEach(async (file, index) => {
        const filePath = path.resolve(dirPath, file);
        const targetFilePath = path.resolve(targetDirPath, file);
        let readableStream;
        let writableStream;

        let isDir = await this.isDir(filePath);

        //不是目录就执行复制，否者递归
        if (!isDir) {
          // console.log({filePath,dirPath,targetDirPath});
          // 创建读取流
          readableStream = await fs.createReadStream(filePath);
          // 创建写入流
          writableStream = await fs.createWriteStream(targetFilePath);
          // 通过管道来传输流
          await readableStream.pipe(writableStream);
          let timer = setTimeout(() => {
            this.loading = false;
            shell.showItemInFolder(targetDirPath);
            clearTimeout(timer);
          }, 1000);
        } else {
          this.moveFiles(filePath, targetDirPath);
        }
      });
    },

    async startTrans(dirPath) {
      this.loading = true;
      const targetDirPath = path.resolve(dirPath, "../target");
      const parentPath = path.dirname(dirPath);
      const files = await this.getAllFiles(parentPath);
      // console.log(files);
      if (!files.includes("target")) {
        await fsp.mkdir(targetDirPath);
        await this.moveFiles(dirPath, targetDirPath);
      } else {
        await this.moveFiles(dirPath, targetDirPath);
      }
    }
  }
};
</script>

<style lang="less" scoped>
.home {
  -webkit-app-region: no-drag;
  width: 100%;
  height: 100vh;
  label {
    -webkit-app-region: no-drag;
    cursor: pointer;
    display: block;
    text-align: center;
    color: #fff;
    background-color: transparent;
    border-radius: 4px;
    height: 100px;
    width: 60%;
    border: 2px dashed #e0dfd5;
    position: relative;
    margin: 0 auto;
    margin-top: 30px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  [type="file"] {
    width: 0.1px;
    height: 0.1px;
    opacity: 0;
    overflow: hidden;
    position: absolute;
    z-index: -1;
  }
  .close {
    line-height: 1em;
    font-size: 16px;
    padding: 10px;
    position: absolute;
    top: 0;
    right: 0;
    cursor: pointer;
    font-style: normal;
  }
  .filename {
    width: 90%;
    font-size: 16px;
    line-height: 100px;
    margin: 0 auto;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .button {
    -webkit-app-region: no-drag;
    position: relative;
    height: 50px;
    width: 200px;
    top: 50px;
    left: calc(50vw - 100px);
    background-image: none;
    border: 1px dashed #fff;
    outline: none;
    background-color: transparent;
    border-radius: 1px;
    color: white;
    text-transform: uppercase;
    font-size: 20px;
    letter-spacing: 2px;
    cursor: pointer;
    transition: all 0.2s ease-out;
    &:hover {
      border-color: rgb(91, 75, 95);
      color: rgb(91, 75, 95);
      border-radius: 16px 4px;
    }
  }

  .button::after {
    content: "";
    display: block;
    position: absolute;
    width: 160px;
    height: 40px;
    background-color: black;
    z-index: -1;
    left: calc(50% - 80px);
    top: 10px;
    opacity: 0.3;
    filter: blur(5px);
    transition: all 0.2s ease-out;
  }
  .button:hover::after {
    opacity: 0.5;
    filter: blur(20px);
    transform: translatey(10px) scalex(1.2);
  }
  .loading {
    border-radius: 50px;
    width: 50px;
    left: calc(50vw - 25px);
  }
  .button.loading::after {
    width: 40px;
    left: 5px;
    top: 12px;
    border-radius: 100%;
  }
  .spinner {
    display: inline-block;
    width: 30px;
    height: 30px;
    position: absolute;
    top: 10px;
    left: 20px;
    background: transparent;
    box-sizing: border-box;
    border-top: 4px solid white;
    border-left: 4px solid transparent;
    border-right: 4px solid transparent;
    border-bottom: 4px solid transparent;
    border-radius: 100%;
    animation: spin 0.6s ease-out infinite;
  }
  @keyframes spin {
    100% {
      transform: rotate(360deg);
    }
  }
}
</style>
