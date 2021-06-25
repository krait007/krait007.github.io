## install electron

```bash
npm install electron -g
```
### 下载安装electron安装慢的问题解决

```bash
cd /usr/local/lib/node_modules/electron
vi node_modules/@electron/get/dist/cjs/artifact-utils.js

const BASE_URL = 'https://github.com/electron/electron/releases/download/';
修改为taobao镜像下载地址
const BASE_URL = 'https://npm.taobao.org/mirrors/electron/';

再执行
npm install electron -g

```

```
I found a way in Chinese region.

Step 1. npm install electron
Step 2. download the electron zip from https://github.com/electron/electron/releases/download/v7.1.7/electron-v7.1.7-darwin-x64.zip
Step 3. copy the zip to /electron/dist
Step 4. vi ./node_modules/electron/path.txt and input /electron-v7.1.7-darwin-x64/Electron.app/Contents/MacOS/Electron

Finaly, you can run npm start.


```

## 镜像配置

```
# 指定 npm 国内镜像
$ npm config set registry=https://registry.npm.taobao.org/
# 指定 Electron 的国内镜像地址
$ npm config set ELECTRON_MIRROR=https://npm.taobao.org/mirrors/electron/ 
$ npm install

```



## Puppeteer

```bash
export PUPPETEER_DOWNLOAD_HOST=https://npm.taobao.org/mirrors

#跳过 Chromium 内核的下载
export PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=1

```