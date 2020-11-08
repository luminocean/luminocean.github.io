---
title: ' Webpack: 从0到1'
date: "2019-11-08T00:01:07+08:00"
tags:
- frontend
draft: false
---

# 0. 前言
说实话，我不太喜欢Webpack。

作为一个从0.12时代开始使用Node.js的用户来说，我一直都很喜欢（或习惯于）CommonJS风格的模块组织风格。于是当我见到Browserify的时候，不禁惊叹于这个将Node.js原封不动就可以挪到前端的想法，机智又简单。加上Gulp的工具链支持，算得上一个很漂亮的前端构建方案。

但是紧随其后出现的Webpack以其强大的功能以及社区支持在近几年迅速赶超，俨然已成为前端构建工具的事实标准。对于这一点我还是有一点失落的。更让我失落的是，Webpack却似乎没有一个当红工具所应有的易用性。每次使用Webpack都需要查文档，而且每次在查了文档以后都有一种死记硬背的感觉。略带讽刺的是，居然还出现了[Create App - your tool for starting a new webpack or Parcel project](https://createapp.dev/webpack)这样的的网站，某种程度上Webpack配置的复杂性可见一斑。顺带一句，Parcel这样标榜零配置的项目的出现不知算不算是对Webpack此类配置繁琐工具的一种反击？

不管怎样，在未来的两到三年内我相信Webpack依然会在前端构建领域占据统治地位。即便未来并非如此，Webpack也会是一个重新审视前端构建流程的绝好样本。这也是我写作本文的初衷。这不是另一篇（yet another）所谓的「Webpack教程」，而是一篇希望能从需求和设计思路出发试图诠释并理解Webpack的文章，也希望能对读者有一点启发或者借鉴意义。

# 1. Why Webpack?
不管使用怎样的构建工具，最终构建完成的前端项目无非还是主要三个元素：HTML, Javascript和CSS（当然，也存在着诸如ASM.js或者Wasm之类的技术，当然也可以认为是Javascript的变体）。而这些元素最终都是以最古典的方式组织起来的：HTML引用js和css，再由浏览器进行获取与渲染。但是如果我们也使用这类原始的组织方式的话，有很多问题无法很好的解决，比如：
* 多个js文件都由HTML来引用，共享一个全局空间，存在着js文件之间依赖关系混乱的问题
* 如果要使用Typescript或者SCSS之类的上层工具，还需要专门维护源文件到目标文件的编译关系
* 使用第三方库依然不方便
通过使用Webpack，我们可以把很多任务进行一站式地处理。一旦配置完成以后，构建任务也就是一行命令的事儿。

# 2. 示例项目
由于我们将使用npm管理所有的工具链，我们先从创建一个空的npm项目开始：
```bash
# terminal
mkdir webpack-playground
cd webpack-playground
npm init -y
```

添加一些文件以后，项目结构如下：
```
- dist (empty)
- node_modules
- src
  - app.js
  - hello.js
- index.html
- package.json
- package-lock.json
- webpack.config.js
```
其中：
```html
<!-- ./index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Webpack!</title>
</head>
<body>
    <div id="root"></div>
    <!-- this path can be served by either a dev server 
        or production static server -->
    <script src="/dist/bundle.js"></script>
</body>
</html>
```
```javascript
// ./src/hello.js
export function sayHello() {
    alert("Hello!")
}
```
```javascript
// ./src/app.js
// use {} to wrap non-default exports
import { sayHello } from './hello'

const root = document.getElementById('root');
const button = document.createElement('button');
button.textContent = 'Click';
button.onclick = () => {
    sayHello();
}
root.append(button);
```
和传统的结构相比，在这里我们做了两点改动：
* HTML中引用了`/dist/bundle.js`这个不存在的文件。我们将在之后构建这个文件。
* JS文件被拆成了两个，之间通过ES6 Module的语法组织起来。

为了把这个项目跑起来，我们尝试使用Webpack。首先安装：
```bash
npm install webpack webpack-cli webpack-dev-server --save-dev
```
其中，`webpack`是webpack的基础库。而`webpack-cli`则封装了webpack的CLI工具，使得我们可以使用`webpack`命令来构建项目。

接下来我们对Webpack做一点基础配置：
```javascript
// regular Node.js module
const path = require('path');

module.exports = {
    entry: './src/app.js',
    output: {
        // webpack requires this to be an absolute path
        path: path.resolve(__dirname, 'dist'),
        filename: 'bundle.js',
        // indicates how to render the resource directory in HTML files.
		  // webpack-dev-server may also read this field.
        publicPath: 'dist/'
    },
    // to suppress the missing mode warning 
    mode: 'none'
}
```

然后我们就可以在项目根目录下使用`webpack`命令来构建了。但是由于我们使用的是局部安装，`webpack`可执行文件被安装到了`node_modules/.bin`里面，因此直接执行命令是找不到的。我们当然可以直接执行`./node_modules/.bin/webpack`，但是为了简洁和方便起见，可以把命令放到`package.json`里面：
```javascript
// ...
"scripts": {
    "build": "webpack -p",
    "dev": "webpack-dev-server"
},
// ...
```
执行`npm run build`即可构建，产物`bundle.js`在`./dist`里面。`-p`会对产物进行压缩。
使用`npm run dev`的话会启动一个开发服务器，产物会被放在内存中。Dev Server会读取配置中的`output.publicPath`来响应浏览器的资源请求，如果匹配则中内存中取出结果并返回。当然`output.publicPath`可能也会被其它Webpack的组件所读取，此处暂不讨论。

启动Dev Server以后，访问`localhost:8080`即可验证功能的完整性，可见Webpack已经完成了构建。或者也可以build以后单独起一个static server或者干脆`python -m http.server 8080`来查看结果。

# 3. 使用图片/SCSS/ES6
### 3.1 Import 图片
现在才刚刚开始。接下来我们会尝试引入一些更高级的资源。首先，我们来加一张图片：
```bash
mkdir -p assets
# Author page: [wlop | DeviantArt](https://www.deviantart.com/wlop)
curl -L https://bit.ly/2Xr5dHE > assets/scene.jpg
```

为了使用这张图片，我们需要使用一个`<img>`并把其`src`指向一个正确的位置。Webpack支持一种反直觉的处理方式：把图片作为一个资源import进js文件中。如下：
```javascript
// ./src/app.js
import scenePic from '../assets/scene.jpg';
// ...
const img = document.createElement('img');
img.alt = 'Scene';
// in case we need to add some styling to the image
img.classList.add('scene');
// render image link at compile time
img.src = scenePic;
root.appendChild(img);
```
可以看到，引入的`scenePic`其实只是一个字符串（完成后面的配置并构建后，可以发现这个值形如`"dist/5839523a9d082a8f4ce41c9db651c870.jpg"`，其中`dist/`部分就是我们之前配的`publicPath`字段）。

显然Javascript本身不可能支持这样的语法，我们需要一个Webpack插件来实现这个效果。安装：
```bash
npm install file-loader --save-dev
```
并配置Webpack:
```javascript
// ./webpack.config.js
// ...
module: {
        rules: [
            {
                test: /\.(jpg|png)$/,
                use: ['file-loader']
            }
        ]
    }
// ...
```
这里的配置指示Webpack在处理到jpg的import操作时，使用file-loader来处理。而file-loader则会根据路径找到图片文件，计算其hash值，将其与`publicPath`字段合并后的字符串返回作为import后的默认值。

Build并验证效果。

### 3.2 Import SCSS

接下来我们尝试加入SCSS：
```css
// ./src/app/scss
.scene {
    width: 200px;
}
```
```javascript
// ./src/app.js
import './app.scss'
```
安装对应SCSS loaders：
```bash
npm install sass-loader node-sass style-loader css-loader --save-dev
```
其中node-sass是用来实际编译SCSS到CSS的工具。最后配置Webpack:
```javascript
// ./webpack.config.js
// ...
module: {
        rules: [
            // ...
            {
                test: /\.scss$/,
                use: [
                    'style-loader', // creates style nodes from JS strings
                    'css-loader', // load CSS as strings
                    'sass-loader' // compiles Sass to CSS, using Node Sass by default
                ]
            }
        ]
    }
// ...
```

有趣的是这里一口气配了三个loader，各司其职。

Build并验证效果。

### 3.3 Import JS新标准

添加如下配置，使用babel-loader即可在load任何js文件的时候进行文件转换，从而保证新特性一直可用：
```javascript
// ./webpack.config.js
// ...
module: {
    rules: [
    // ...
        {
            test: /\.js$/,
            exclude: /node_modules/,
            use: {
                loader: 'babel-loader',
                options: {
                    presets: ['@babel/preset-env']
                }
            }
        }
    ]
}
// ...
```

