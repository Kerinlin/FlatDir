# Electron+Vue+Node 实现展平文件夹小工具

![GIF.gif](https://i.loli.net/2020/12/25/qKiAN9JvUWV2moE.gif)

需求起源是对象整理音效的时候，她收集的音效资源是有嵌套的子文件夹的，但是她想把所有的文件都提到一级目录，没有程序之前，她的操作是：

1. 复制子文件夹中的文件到一级目录
2. 删除子文件夹

如果子文件夹少，那工作量也还算不大，但是如果子文件夹嵌套较深，这工作量就上来了，而且都是重复的工作。于是她过来寻求我的帮助，我一看刚好自己最近不是在学习node吗，这刚好可以练习下node。

## 实现流程
![1.png](https://i.loli.net/2020/12/29/Vlz4dypROBS6IFW.png)

实现流程主体是一个递归，遍历文件夹，如果是文件就执行转移操作，不是就传入新的文件夹路径，继续遍历。

## 具体实现
具体实现分以下几步：
1. 项目初始化
2. 获取文件路径,以及路径下所有文件
3. 转移操作
4. 删除操作

#### 项目初始化

因为使用平台为windows，然后又不能把界面做丑，所以技术选择为Electron + Node。虽然electron打包出来文件有 50M,但是软件带来的收益是远远大于需要付出的代价的。对于electron与vue的集成，之前写过文章 [Electron+vue从零开始打造一个本地音乐播放器](https://juejin.cn/post/6887964816237920263)。electron与vue集成目前社区有两个方案:
1. [SimulatedGREG/electron-vue](https://github.com/SimulatedGREG/electron-vue),这个比较老，比较臃肿，不过如果是老项目的迁移的话，可以考虑使用。
2. [nklayman/vue-cli-plugin-electron-builder](https://github.com/nklayman/vue-cli-plugin-electron-builder),这个支持最新稳定版本的electron以及最新的vue脚手架，代码里主进程相关代码集成在background.js中，其他代码组织跟写vue项目没有区别。[文档](https://nklayman.github.io/vue-cli-plugin-electron-builder/)也特别棒，所以比较推荐使用这个。

这里我弄了一个很简单的项目初始化模板，集成最新稳定版本的electron,以及最新的vue-cli4,另外还添加了normalize.css初始化样式。electron+vue相关项目的初始化可以直接使用这个模板,[戳这里](https://github.com/Kerinlin/simple-electron-vue-template)。

#### 获取文件路径,以及路径下所有文件

**获取文件路径**，这里的处理方式，跟之前的翻译项目([Electron+Vue从零开始打造一个本地文件翻译器](https://juejin.cn/post/6899581622471884813))一样，通过两种方式获取到需要的路径。

1. 设置input webkitdirectory directory 属性，然后监听change事件获取到所选择文件夹的路径
2. 通过H5的拖拽API监听drop事件，获取到 DataTransfer 对象，DataTransfer 对象用于保存拖动并且放下的数据。

这里不一样的是需要对拖入的文件进行判断，必须拖入的是文件夹。

          const originFiles = [...e.dataTransfer.files];
          const isAllDir = originFiles.every(file =>
            fs.statSync(file.path).isDirectory()
          );

          if (!isAllDir) {
            ipcRenderer.send("confirmDialog");
            return false;
          }

主进程

    ipcMain.on("confirmDialog", () => {
      dialog.showMessageBox({
        type: "info",
        title: "确认",
        message: "请确认选择的文件是否都是文件夹"
      });
    });

**获取路径下所有的文件**

        // 获取文件路径下的所有文件
        async getAllFiles(path) {
          try {
            const res = await fsp.readdir(path);
            return res;
          } catch (error) {
            console.log(error);
          }
        },

#### 转移操作

转移操作这里使用的是fs模块的rename方法，需要判断是否是文件夹来决定是否需要递归操作。对于重名文件,会判断待转移文件与目标文件夹文件大小是否一致，如果不一致会重新命名。重新命名需要生成一个文件id附带文件名称在后面，保证文件不会重名。具体代码如下：

**判断是否是文件夹**

        // 判断是否是文件夹
        async isDir(path) {
          try {
            const res = await fsp.stat(path);
            if (res.isDirectory()) {
              return true;
            } else {
              return false;
            }
          } catch (error) {
            console.log(error);
          }
        },


**生成文件id**

        // 生成文件id
        uuid(len, radix) {
          const chars = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'.split('');
          let uuid = [],
            i;
          radix = radix || chars.length;

          if (len) {
            // Compact form
            for (i = 0; i < len; i++) uuid[i] = chars[0 | (Math.random() * radix)];
          } else {
            // rfc4122, version 4 form
            var r;

            // rfc4122 requires these characters
            uuid[8] = uuid[13] = uuid[18] = uuid[23] = '-';
            uuid[14] = '4';

            // Fill in random data. At i==19 set the high bits of clock sequence as
            // per rfc4122, sec. 4.1.5
            for (i = 0; i < 36; i++) {
              if (!uuid[i]) {
                r = 0 | (Math.random() * 16);
                uuid[i] = chars[i == 19 ? (r & 0x3) | 0x8 : r];
              }
            }
          }

          return uuid.join('');
        },

**转移操作**

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

          console.log('遍历了所有的文件');
          console.log({ filePath, targetFilePath });

          if (fs.existsSync(targetFilePath)) {
            console.log(`目标文件 ${file} 存在于目标文件夹 ${targetDirPath} 中`);
            console.log({ filePath, targetFilePath });

            const targetFiles = await this.getAllFiles(targetDirPath);
            const fileIndex = targetFiles.indexOf(file);

            console.log(`同名文件 ${file} 的下标为 ${fileIndex}`);

            // 获取两边文件的信息，进行对比
            const targetFileInfo = await fsp.stat(targetFilePath);
            const originFileInfo = await fsp.stat(filePath);

            console.log(`原文件与目标文件夹文件的大小分别为 ${originFileInfo.size}  ${targetFileInfo.size}`);
            // 如果有重名文件，判断文件大小是否一致
            if (fileIndex >= 0 && originFileInfo.size !== targetFileInfo.size) {
              // 获取原文件的名称以及后缀格式
              const fileExt = path.extname(filePath);
              const fileName = path.basename(filePath, fileExt);

              // 生成新的文件名称
              const newFileName = `${fileName}-${this.uuid(6, 16)}`;
              const newPath = path.resolve(dirPath, `${newFileName}${fileExt}`);
              const newTargetFilePath = path.resolve(targetDirPath, `${newFileName}${fileExt}`);

              // 重命名
              await fsp.rename(filePath, newPath);

              // 移动至新目标文件夹
              await fsp.rename(newPath, newTargetFilePath);
            } else {
              console.log('目标文件同名但是文件内容不一样');
              console.log({ filePath, targetFilePath });
              // 不是同名文件就直接复制移动
              await fsp.rename(filePath, targetFilePath);
            }
          } else {
            console.log('目标文件不存在于目标文件夹中');
            await fsp.rename(filePath, targetFilePath);
          }
        } else {
          // 是目录，执行递归操作
          await this.moveFiles(filePath, targetDirPath);
        }
        let timer = setTimeout(async () => {
          await this.removeDir(dirPath);
          this.loading = false;
          clearTimeout(timer);
        }, 1000);
      });
    },

#### 删除操作
由于执行完复制移动方法后原文件夹还存在，所以需要删除。删除操作也是需要查询是否是文件夹，如果是文件就执行fs.unlinkSync方法，是目录就进行递归，最后删除完，保证了仅剩下目录，然后通过fs.rmdirSync，执行删除目录的操作。具体代码如下

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
            console.log('已删除文件');
          }
        },


## 最后
最近在对象面前装逼太多了，在对象眼里我都是发光的存在，她就跟她的同学讲程序猿有多神奇(虽然在咱们眼里是小事)，然后她的同学们**都纷纷说要找程序猿男友**，我心里暗喜，我终于为咱们程序猿争光了！！！源码在这 [戳这里](https://github.com/Kerinlin/FlatDir "戳这里")