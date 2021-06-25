## Spectron

建立在 ChromeDriver 和 WebDriverIO 之上



### 安装

```
yarn add -D spectron mocha
yarn add -D electron-test
yarn add puppeteer-core    #不带浏览器
#yarn add puppeteer   #自带浏览器
```



### macos下查看executablePath

通过 浏览器访问chrome://version/  可以查看executablePath

executablePath: '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome'



### puppeteer方式执行test

```
const puppeteer = require('puppeteer');
(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto('https://www.bing.com');
  await page.screenshot({ path: 'bing_com.png' });

  await browser.close();
})();
```



### puppeteer-core方式执行test

需要指定executablePath

```
const puppeteer = require('puppeteer-core');
(async () => {
  const browser = await puppeteer.launch({
    headless: false,
    executablePath: '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome',
    args: ["--app-shell-host-window-size=1600x1239"]
  });
  const page = await browser.newPage();
  await page.goto('https://www.bing.com');
  await page.screenshot({path: 'bing_com.png'});

  await browser.close();
 })();

```



### Electron + Puppeteer (+ Jest optionally)

https://github.com/peterdanis/electron-puppeteer-demo

 Electron using `puppeteer-core` package is recommended instead of standard `puppeteer` package, as the `-core` version does not download Chromium by default.

To connect Electron and Puppeteer together you have to start Electron app yourself (via `child_process.spawn`)

```
const electron = require("electron");
const puppeteer = require("puppeteer-core");
const { spawn } = require("child_process");
const port = 9200;

spawn(electron, [".", `--remote-debugging-port=${port}`], { shell: true });
```

...and then connect to it via `puppeteer.connect` method

```
(async () => {
  await puppeteer.connect({
        browserURL: `http://localhost:${port}`,
        defaultViewport: { width: 1000, height: 600 }
      });
})()
```



### 问题

```
1. Error: Could not find browser revision 818858. Run "PUPPETEER_PRODUCT=firefox npm install" or "PUPPETEER_PRODUCT=firefox yarn install" to download a supported Firefox browser binary.

# 检查使用的是否为puppeteer-core包
const puppeteer = require('puppeteer-core');
->
executablePath: '...'
or
const puppeteer = require('puppeteer');




```



