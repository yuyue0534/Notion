`package.json` 是：

> 整个 Node.js / 前端工程 的“项目身份证 + 启动控制中心 + 依赖管理中心”。

你可以理解成：

```text
Linux里的：
Makefile + requirements.txt + 项目元数据

前端里的：
npm控制台 + 工程配置总入口
```

Vue / React / Next / Electron / NestJS……

所有现代 JS 项目：

都离不开它。

---

# 一、先看一个真实的 package.json

这是一个典型 Vue3 + Vite 项目：

```json id="7cw7wa"
{
  "name": "vue-admin-system",
  "version": "1.0.0",
  "private": true,

  "type": "module",

  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },

  "dependencies": {
    "vue": "^3.5.0",
    "vue-router": "^4.4.0",
    "pinia": "^2.3.0",
    "axios": "^1.8.0"
  },

  "devDependencies": {
    "vite": "^6.0.0",
    "@vitejs/plugin-vue": "^5.2.0",
    "typescript": "^5.7.0",
    "sass": "^1.83.0"
  }
}
```

但实际上：

这只是最基础形态。

大型项目的 package.json 往往几百行。

---

# 二、package.json 本质是什么

它本质是：

```text
Node项目的元数据配置文件
```

Node/npm 会读取它。

---

# 三、name 项目名称

```json id="h3ks6x"
"name": "vue-admin-system"
```

---

## 作用

---

### 1）npm 包名称

发布 npm 时：

```bash id="2dnh07"
npm install vue-admin-system
```

---

### 2）node_modules 标识

---

### 3）日志/依赖树显示

---

# 四、version 版本号

```json id="j8n9st"
"version": "1.0.0"
```

遵循：

```text
major.minor.patch
```

即：

```text
大版本.功能版本.修复版本
```

---

## 举例

---

### 1.0.0 → 1.0.1

修复 bug。

---

### 1.0.0 → 1.1.0

新增功能。

---

### 1.0.0 → 2.0.0

破坏性更新。

API 不兼容。

---

# 五、private

```json id="2hzw92"
"private": true
```

极其重要。

---

## 作用

防止：

```bash id="i0a92u"
npm publish
```

误发布。

很多公司内部项目都必须：

```json id="gvz69n"
"private": true
```

---

# 六、type: module（现代 Node 核心）

```json id="8jczrl"
"type": "module"
```

这个非常关键。

---

# 七、CommonJS vs ESModule

Node 曾经只有：

```js id="2qxbn8"
const a = require('a')
module.exports = {}
```

即：

```text
CommonJS
```

---

后来 JS 官方推出：

```js id="nmtjlwm"
import a from 'a'
export default {}
```

即：

```text
ESModule
```

---

## type=module 的作用

告诉 Node：

```text
整个项目使用 ESModule
```

于是：

```js id="y08h91"
import path from 'path'
```

合法。

否则会报错。

---

# 八、scripts（最重要配置之一）

核心中的核心。

---

```json id="lc4a7d"
"scripts": {
  "dev": "vite",
  "build": "vite build",
  "preview": "vite preview"
}
```

---

# 九、scripts 本质是什么

其实：

```bash id="8h7tql"
npm run xxx
```

等价于：

```bash id="ug7s1j"
执行一段 shell 命令
```

---

## 举例

---

### 运行：

```bash id="c0ipj5"
npm run dev
```

实际上：

```bash id="c41c2v"
vite
```

---

### 运行：

```bash id="68ycvq"
npm run build
```

实际上：

```bash id="g4o4gx"
vite build
```

---

# 十、为什么能直接执行 vite

因为：

```text
node_modules/.bin
```

会自动加入 PATH。

---

即：

安装：

```bash id="yvw4ut"
npm install vite
```

后：

```bash id="g59o0l"
node_modules/.bin/vite
```

存在。

npm run 自动找到它。

---

# 十一、企业级 scripts 长什么样

真实项目：

```json id="wdl97p"
"scripts": {
  "dev": "vite",

  "build:test": "vite build --mode test",

  "build:prod": "vite build --mode production",

  "lint": "eslint src --fix",

  "format": "prettier --write .",

  "type-check": "vue-tsc --noEmit",

  "commit": "git-cz",

  "prepare": "husky install"
}
```

---

# 十二、dependencies vs devDependencies

这是前端新人最大误区之一。

---

# 十三、dependencies

```json id="tdtcmv"
"dependencies": {
  "vue": "^3.5.0",
  "axios": "^1.8.0"
}
```

表示：

```text
生产环境也需要
```

---

## 为什么

因为：

浏览器运行时：

需要：

* vue
* axios
* pinia

---

# 十四、devDependencies

```json id="p7j5ec"
"devDependencies": {
  "vite": "^6.0.0",
  "typescript": "^5.7.0"
}
```

表示：

```text
开发时才需要
```

---

## 例如

浏览器根本不需要：

* vite
* eslint
* prettier
* typescript

这些只是：

> 开发工具。

---

# 十五、为什么区分这两个

因为：

生产部署时：

```bash id="tfxbg0"
npm install --production
```

不会安装 devDependencies。

节省空间。

---

# 十六、版本号里的 ^ 到底什么意思

这个极其重要。

---

## 举例

```json id="hmd24z"
"vue": "^3.5.0"
```

意思：

允许：

```text
3.x.x
```

自动更新。

但：

```text
4.x.x
```

不允许。

---

# 十七、常见符号

---

## ^

允许次版本更新：

```text
3.5.0 → 3.9.9
```

---

## ~

只允许 patch：

```text
3.5.0 → 3.5.9
```

---

## 无符号

固定版本。

---

## *

任何版本。

危险。

---

# 十八、node_modules 为什么这么巨大

因为：

依赖树。

---

## 举例

你安装：

```bash id="z7v5yt"
vite
```

实际上：

```text
vite
  ↳ rollup
      ↳ chokidar
      ↳ esbuild
      ↳ ...
```

几十上百层依赖。

---

# 十九、package-lock.json 是什么

很多人误删它。

实际上非常重要。

---

# 二十、为什么需要 lock 文件

否则：

今天安装：

```text
axios 1.8.0
```

明天：

```text
axios 1.9.0
```

可能：

项目炸了。

---

## lock 文件作用

锁死：

```text
精确依赖树
```

确保：

团队所有人安装结果一致。

---

# 二十一、npm install 背后发生了什么

实际上：

---

## 第一步

读取：

```bash id="fij58j"
package.json
```

---

## 第二步

解析依赖树。

---

## 第三步

下载 tar 包。

---

## 第四步

解压：

```bash id="8k8c4w"
node_modules
```

---

## 第五步

生成：

```bash id="6gq1mz"
package-lock.json
```

---

# 二十二、peerDependencies（高级核心）

这个极其容易绕晕。

---

# 二十三、为什么需要 peerDependencies

举例：

你开发：

```text
vue插件
```

比如：

```text
element-plus
```

---

它依赖：

```text
vue
```

但：

它不能自己安装一份 vue。

否则：

项目里会出现：

```text
两个 Vue
```

直接炸。

---

所以：

```json id="6lcbml"
"peerDependencies": {
  "vue": "^3.0.0"
}
```

意思：

```text
宿主项目必须自己安装 vue
```

---

# 二十四、engines

```json id="4jlr0d"
"engines": {
  "node": ">=18"
}
```

限制：

Node 版本。

否则：

低版本可能无法运行。

---

# 二十五、browserslist

前端构建重要配置。

---

```json id="rrmffn"
"browserslist": [
  "> 1%",
  "last 2 versions",
  "not dead"
]
```

作用：

决定：

```text
兼容哪些浏览器
```

---

## Babel / Autoprefixer 会读取它

例如：

自动加：

```css id="jlwmxq"
-webkit-
```

---

# 二十六、bin（CLI工具核心）

如果你开发 CLI：

```json id="1xjz5d"
"bin": {
  "my-cli": "./bin/index.js"
}
```

安装后：

```bash id="jlwmkj"
my-cli
```

直接能运行。

---

# 二十七、exports（现代 Node 核心）

非常重要。

---

## 旧时代

别人：

```js id="aodxql"
import xxx from 'your-lib/dist/index.js'
```

能乱访问内部文件。

---

## exports

```json id="g0mbgc"
"exports": {
  ".": "./dist/index.js"
}
```

限制：

只能访问公开 API。

---

# 二十八、files

发布 npm 时：

```json id="f7ifpv"
"files": [
  "dist"
]
```

只发布：

```text
dist
```

否则：

源码、测试文件全上传。

---

# 二十九、sideEffects（tree shaking核心）

```json id="yw8tgd"
"sideEffects": false
```

告诉打包器：

```text
没有副作用
```

未使用代码可删除。

---

## 极大影响打包体积

Vue组件库非常依赖这个。

---

# 三十、package.json 在现代前端中的真实地位

它实际上是：

| 功能              | 本质           |
| --------------- | ------------ |
| scripts         | 项目启动入口       |
| dependencies    | 运行时依赖        |
| devDependencies | 开发工具链        |
| engines         | Node版本控制     |
| exports         | 模块边界         |
| browserslist    | 浏览器兼容        |
| bin             | CLI系统        |
| sideEffects     | tree shaking |
| type            | 模块规范         |

---

# 三十一、现代前端工程启动全过程

你执行：

```bash id="fgt8n0"
npm run dev
```

实际上：

---

## npm

读取：

```text
package.json
```

---

## 找到

```json id="7p0j5s"
"dev": "vite"
```

---

## npm PATH注入

找到：

```text
node_modules/.bin/vite
```

---

## Node执行vite

---

## vite读取

```text
vite.config.ts
```

---

## 启动开发服务器

---

## 浏览器加载项目

---

# 三十二、为什么 package.json 越来越复杂

因为：

现代前端已经不只是：

```html id="uy0cpi"
html + css + js
```

而是：

```text
Node生态
编译系统
模块系统
打包系统
CI/CD
Lint
测试
发布
Monorepo
自动化工程
```

package.json 成了：

> 整个 JS 工程生态的控制中枢。

---

# 三十三、真正建议你深入研究的部分（重点）

如果想真正理解现代前端工程：

优先深挖：

---

## 第一梯队（最重要）

* scripts
* dependencies
* lock文件
* type module
* 环境变量

---

## 第二梯队

* peerDependencies
* exports
* sideEffects
* browserslist

---

## 第三梯队（工程化）

* workspace
* monorepo
* changesets
* semantic-release

---

# 三十四、下一步该学什么（最佳路线）

建议你继续深入：

1. npm / pnpm 原理
2. node_modules 为什么扁平化
3. ESModule vs CommonJS
4. Vite 启动原理
5. Rollup 打包原理
6. Tree Shaking
7. Babel
8. Monorepo
9. pnpm workspace
10. CI/CD 自动发布

这时候你会真正理解：

> “现代前端工程化”到底是什么。
