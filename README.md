# 关于React App如何部署在子目录下的情况说明

首先npx create-react-app app然后把build出来的内容ln -s到/var/www/foo/bar/app
然后配置nginx
```
server {
  listen 80;
  server_name space.io;
  root /var/www/foo;

  location / {
    index index.html;
  }
}
```
浏览器访问http://space.io/bar/app，显示报错，无法获取
```
[Error] 404 (Not Found) (http://space.io/static/css/main.e977c7a5.chunk.css)
[Error] 404 (Not Found) (http://space.io/static/js/2.567b0d4d.chunk.js, line 0)
[Error] 404 (Not Found) (http://space.io/static/js/main.872637aa.chunk.js, line 0)
[Error] 404 (Not Found) (http://space.io/favicon.ico, line 0)
```
显然我们的css/js都在http://space.io/bar/app/而非http://space.io，这里出现了第一次mismatch.

通过curl可以发现返回的html内容如下:
```
<!doctype html>
  <html lang="en">
  <head>
  <meta charset="utf-8"/>
  <link rel="shortcut icon" href="/favicon.ico"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <meta name="theme-color" content="#000000"/>
  <meta name="description" content="Web site created using create-react-app"/>
  <link rel="apple-touch-icon" href="logo192.png"/>
  <link rel="manifest" href="/manifest.json"/>
  <title>React App</title>
  <link href="/static/css/main.e977c7a5.chunk.css" rel="stylesheet">
  </head>
  <body>
  <noscript>You need to enable JavaScript to run this app.</noscript>
  <div id="root">
  </div>
  <script>...</script>
  <script src="/static/js/2.567b0d4d.chunk.js"></script>
  <script src="/static/js/main.872637aa.chunk.js"></script>
  </body>
  </html>
```
这个就是build下的index.html，原始文件是app/public/index.html
```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="shortcut icon" href="%PUBLIC_URL%/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#000000" />
    <meta name="description" content="Web site created using create-react-app" />
    <link rel="apple-touch-icon" href="logo192.png" />
    <link rel="manifest" href="%PUBLIC_URL%/manifest.json" />
    <title>React App</title>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
  </body>
</html>
```
这里可以看到%PUBLIC_URL%, 可以理解为默认情况直接替换为'/', 如果需要替换就需要修改package.json中的homepage
```
"homepage": "."
```
重新yarn build后, 页面就OK了, 你可以看到之前的'/'改为'./', 
```
<script src="./static/js/2.567b0d4d.chunk.js"></script>
```
从浏览器中访问，会基于当前path发出请求，即./可以翻译为http://space.io/bar/app/static/js/xxx, 看上去一切ok。

但是加上router后就会有问题,
```
import { BrowserRouter as Router, Route, Link } from "react-router-dom";

function Index() {
  return <h2>Home</h2>;
}

function About() {
  return <h2>About</h2>;
}

function App() {
  return (
    <Router>
      <div>
        <nav>
          <ul>
            <li>
              <Link to="/">Home</Link>
            </li>
            <li>
              <Link to="/about/">About</Link>
            </li>
          </ul>
        </nav>

        <Route path="/" exact component={Index} />
        <Route path="/about/" component={About} />
      </div>
    </Router>
  );
}
```
现在访问http://space.io/bar/app/
能render出li, 但是不能匹配/, 因为页面得到的window.location.pathname的/bar/app/
点击<link to="/">build出来后的href是http://space.io/
所以你要告诉react-router你的基础路径是什么，这时候就需要用到router的basename

## 第一个里程碑
到这里出现了第一个里程碑，就是通过:
- "homepage": "."
- router basename="/bar/app/"

可以正常访问http://space.io/bar/app/, 同时, 点link也都能正确路由, 浏览器上的url也ok。

到这里有2个问题:
1. 如果直接复制这个链接或者刷新http://space.io/bar/app/about/就直接显示404了
2. router的basename写死了，不能自适应


一个个问题来解决，
首先显示404的原因很简单，因为nginx找不到/bar/app/about这个页面，刚才输入的是/bar/app/所以映射到index.html，但是在/bar/app/about/下没有index，你可以在build里mkdir about实验一下。
这个也容易, 在nginx通过try file就行了。
通过try file后，homepage的.就不能再用了，因为会引导到http://space.io/bar/app/about/static/....，所以也必须写死'/bar/app'

React Router 是建立在 history 之上的，history 监听浏览器地址栏的变化并解析URL转化为location对象，然后router使用它匹配到路由，正确地渲染对应的组件。

## 第二个里程碑
借助后端的nginx配合
```
location / {
  index index.html;
  try_files $uri /bar/app/index.html;
}
```
然后在编译前就要确定部署时的subdirectory, 然后在package.json中配置
```
"homepage": "/bar/app/"
```
vscode还提示我这里homepage要写完整, 但如果要编译时还不知道部署到哪呢?
同时router采用同样的basename,
```
import pkg from "../package.json"
<Router basename={pkg.homepage}>
   ...
</Router>
```

react router v4/v5支持3种router：
- BrowserRouter
- HashRouter
- MemoryRouter

在v4以后，应该来说肯定首推BrowserRouter, 因为看上去更好更自然, 如果用hash, url就会变成http://space.io/bar/app#/about, hash不需要做任何配置，不需要服务器后台介入try_files, 缺点就是不够好看, SEO不友好, 在极端情况还是会有些问题。

<HashRouter>
A <Router> that uses the hash portion of the URL (i.e. window.location.hash) to keep your UI in sync with the URL.IMPORTANT NOTE: Hash history does not support location.key or location.state. In previous versions we attempted to shim the behavior but there were edge-cases we couldn’t solve. Any code or plugin that needs this behavior won’t work. As this technique is only intended to support legacy browsers, we encourage you to configure your server to work with <BrowserHistory> instead.

## 第三个里程碑
权衡利弊，还是BrowserRouter更好些, 麻烦也就麻烦一点, 需要做两件事情:
- 后端: 针对每个app写下location block做对应的try_files
- 前端: 编译前确定hompage, 写下相对路径 (/bar/app/)

如果一定需要编译后决定，就在index.html中增加类似的config
```
<script>
  System.config({ baseURL: '/' });
</script>
```

疑难杂症:
Q: Uncaught SyntaxError: Unexpected token '<'
A: 在这个上下文中就是因为加载的js因为路径原因不存在，又因为try_files的关系给出了一个html, js引擎因为不能解析html的<!doctype html>引起的错误。

参考文档:
[javascript - React-router urls don't work when refreshing or writing manually - Stack Overflow](https://stackoverflow.com/questions/27928372/react-router-urls-dont-work-when-refreshing-or-writing-manually)
[react-router hash history and browser history](https://www.jianshu.com/p/7237bf23db6a)
