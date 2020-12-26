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
const { shell } = window.require("electron");
const { ipcRenderer } = window.require("electron");
export default {
  name: "home",
  data() {
    return {
      path: "",
      loading: false,
      back: false,
      droppedFiles: [],
      isDropMulti: false,
      result: []
    };
  },
  methods: {
    addFile(e) {
      const originFiles = [...e.dataTransfer.files];
      const isAllDir = originFiles.every(file =>
        fs.statSync(file.path).isDirectory()
      );

      if (!isAllDir) {
        ipcRenderer.send("confirmDialog");
        return false;
      }

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

    // 删除文件
    removeDir(url) {
      if (fs.existsSync(url)) {
        const files = fs.readdirSync(url);
        files.forEach((file, index) => {
          const curPath = path.join(url, file);

          if (fs.statSync(curPath).isDirectory()) {
            console.log(`检测到目录 ${curPath}`);
            this.removeDir(curPath);
          } else {
            fs.unlinkSync(curPath);
            console.log(`已删除文件 ${curPath}`);
          }
        });
        // 删除文件夹
        fs.rmdirSync(url);
      } else {
        console.log("已删除文件");
      }
    },

    // 生成uuid
    uuid(len, radix) {
      var chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz".split(
        ""
      );
      var uuid = [],
        i;
      radix = radix || chars.length;

      if (len) {
        // Compact form
        for (i = 0; i < len; i++) uuid[i] = chars[0 | (Math.random() * radix)];
      } else {
        // rfc4122, version 4 form
        var r;

        // rfc4122 requires these characters
        uuid[8] = uuid[13] = uuid[18] = uuid[23] = "-";
        uuid[14] = "4";

        // Fill in random data. At i==19 set the high bits of clock sequence as
        // per rfc4122, sec. 4.1.5
        for (i = 0; i < 36; i++) {
          if (!uuid[i]) {
            r = 0 | (Math.random() * 16);
            uuid[i] = chars[i == 19 ? (r & 0x3) | 0x8 : r];
          }
        }
      }

      return uuid.join("");
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

        let isDir = await this.isDir(filePath);

        //不是目录就执行复制，否者递归
        if (!isDir) {
          // 检查移动目标文件夹是否有重名文件

          console.log("遍历了所有的文件");
          console.log({ filePath, targetFilePath });

          if (fs.existsSync(targetFilePath)) {
            console.log(
              `目标文件 ${file} 存在于目标文件夹 ${targetDirPath} 中`
            );
            console.log({ filePath, targetFilePath });

            const targetFiles = await this.getAllFiles(targetDirPath);
            const fileIndex = targetFiles.indexOf(file);

            console.log(`同名文件 ${file} 的下标为 ${fileIndex}`);

            // 获取两边文件的信息，进行对比
            const targetFileInfo = await fsp.stat(targetFilePath);
            const originFileInfo = await fsp.stat(filePath);

            console.log(
              `原文件与目标文件夹文件的大小分别为 ${originFileInfo.size}  ${targetFileInfo.size}`
            );
            // 如果有重名文件，判断文件大小是否一致
            if (fileIndex >= 0 && originFileInfo.size !== targetFileInfo.size) {
              // 获取原文件的名称以及后缀格式
              const fileExt = path.extname(filePath);
              const fileName = path.basename(filePath, fileExt);

              // 生成新的文件名称
              const newFileName = `${fileName}-${this.uuid(6, 16)}`;
              const newPath = path.resolve(dirPath, `${newFileName}${fileExt}`);
              const newTargetFilePath = path.resolve(
                targetDirPath,
                `${newFileName}${fileExt}`
              );

              // 重命名
              await fsp.rename(filePath, newPath);

              // 移动至新目标文件夹
              await fsp.rename(newPath, newTargetFilePath);
            } else {
              console.log("目标文件同名但是文件内容不一样");
              console.log({ filePath, targetFilePath });
              // 不是同名文件就直接复制移动
              await fsp.rename(filePath, targetFilePath);
            }
          } else {
            console.log("目标文件不存在于目标文件夹中");
            await fsp.rename(filePath, targetFilePath);
          }
        } else {
          // 是目录，执行递归操作
          await this.moveFiles(filePath, targetDirPath);
        }
        let timer = setTimeout(async () => {
          await this.removeDir(dirPath);
          // shell.showItemInFolder(targetDirPath);
          this.loading = false;
          clearTimeout(timer);
        }, 1000);
      });
    },

    getMultiFile(dirPath, files) {},

    async startTrans(dirPath) {
      this.loading = true;
      if (this.isDropMulti) {
        // 拖拽多个文件夹的情况

        // 在同级目录下创建新的目标文件夹
        const targetDirPath = path.resolve(dirPath, "合并");
        const files = await this.getAllFiles(dirPath);
        if (!files.includes("合并")) {
          await fsp.mkdir(targetDirPath, { recursive: true });
        }

        this.droppedFiles.forEach((file, index) => {
          console.log(file);
          const filePath = file.path;
          this.moveFiles(filePath, targetDirPath);
        });
      } else {
        const targetDirPath = path.resolve(dirPath, "../合并");
        const parentPath = path.dirname(dirPath);
        const files = await this.getAllFiles(parentPath);
        // console.log(files);
        if (!files.includes("合并")) {
          await fsp.mkdir(targetDirPath, { recursive: true });
        }
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
