以现在主流的 Vue 3 技术栈来说，`vite.config.ts/js` 基本就是整个前端工程的“总控制台”。
它负责：

* 开发服务器怎么启动
* 路径别名怎么解析
* 打包输出到哪里
* 插件怎么工作
* 环境变量怎么注入
* 代理/API怎么转发
* 性能优化怎么做
* 多环境（开发/测试/生产）怎么区分

你可以把它理解成：

> “Vue 项目的编译器 + 开发服务器 + 打包器”的统一配置入口。

下面我会从“真实项目”的角度，一层层拆解。

---

# 一、一个典型 vite.config.ts 长什么样

先看一个真实项目里常见的配置：

```ts
import { defineConfig, loadEnv } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from 'path'

export default defineConfig(({ mode }) => {

  const env = loadEnv(mode, process.cwd())

  return {
    plugins: [
      vue()
    ],

    resolve: {
      alias: {
        '@': path.resolve(__dirname, './src')
      }
    },

    server: {
      host: '0.0.0.0',
      port: 5173,
      open: true,
      proxy: {
        '/api': {
          target: env.VITE_API_URL,
          changeOrigin: true,
          rewrite: (path) => path.replace(/^\/api/, '')
        }
      }
    },

    css: {
      preprocessorOptions: {
        scss: {
          additionalData: `@import "@/styles/variables.scss";`
        }
      }
    },

    build: {
      outDir: 'dist',
      sourcemap: false,
      minify: 'esbuild',
      rollupOptions: {
        output: {
          manualChunks: {
            vue: ['vue']
          }
        }
      }
    },

    define: {
      __APP_VERSION__: JSON.stringify('1.0.0')
    }
  }
})
```

这其实已经包含了：

* 插件系统
* 别名系统
* dev server
* API代理
* CSS预处理
* 打包优化
* 环境变量
* 全局常量注入

接下来逐块拆。

---

# 二、defineConfig 是什么

```ts
export default defineConfig({
})
```

本质：

```ts
export default {
}
```

也能运行。

但 `defineConfig()` 有两个巨大作用：

---

## 1）TS 类型提示

你写：

```ts
server:
```

IDE 会自动提示：

* host
* port
* proxy
* hmr
* cors

等配置。

否则没有类型推导。

---

## 2）避免配置写错

比如：

```ts
servre: {}
```

拼错了。

defineConfig 会直接提示错误。

---

# 三、plugins 插件系统（核心中的核心）

```ts
plugins: [
  vue()
]
```

Vite 本身只认识 JS。

它并不认识：

* `.vue`
* JSX
* markdown
* svg组件
* 自动导入

所以：

> 一切高级能力，本质都来自插件。

---

## Vue 插件到底做了什么

```ts
vue()
```

实际上：

### 把 `.vue` 文件：

```vue
<template>
</template>

<script setup>
</script>

<style scoped>
</style>
```

转换成：

```js
render()
setup()
css()
```

最终浏览器才能运行。

---

## 常见插件

---

### 自动导入

```ts
AutoImport({
  imports: ['vue']
})
```

于是：

```ts
ref()
computed()
watch()
```

不用 import。

---

### Components 自动注册组件

```ts
Components()
```

于是：

```vue
<HelloWorld />
```

不用：

```ts
import HelloWorld from './xxx'
```

---

### SVG 变组件

```ts
viteSvgIcons()
```

于是：

```vue
<IconHome />
```

就能直接用 svg。

---

### UnoCSS / Tailwind

原子化 CSS。

---

# 四、resolve.alias 路径别名

```ts
resolve: {
  alias: {
    '@': path.resolve(__dirname, './src')
  }
}
```

---

## 为什么需要它

否则：

```ts
import A from '../../../../components/A.vue'
```

地狱。

---

## 有了别名

```ts
import A from '@/components/A.vue'
```

舒服很多。

---

# 五、server 开发服务器

---

## 1）host

```ts
host: '0.0.0.0'
```

允许局域网访问。

否则：

手机无法访问电脑开发服务器。

比如：

```bash
http://192.168.1.5:5173
```

---

## 2）port

```ts
port: 5173
```

开发端口。

---

## 3）open

```ts
open: true
```

启动自动打开浏览器。

---

# 六、proxy 代理（前后端联调核心）

这个极其重要。

---

## 问题来源：跨域

前端：

```bash
localhost:5173
```

后端：

```bash
localhost:8080
```

浏览器认为：

> 不是同一个源。

于是被 CORS 限制。

---

## proxy 如何解决

```ts
proxy: {
  '/api': {
    target: 'http://localhost:8080'
  }
}
```

前端：

```ts
axios.get('/api/user')
```

实际上变成：

```bash
http://localhost:8080/api/user
```

浏览器以为：

> 还是同源。

因为是 Vite server 在中间转发。

---

## rewrite

```ts
rewrite: path => path.replace('/api', '')
```

比如：

前端：

```bash
/api/user
```

转发：

```bash
http://localhost:8080/user
```

---

# 七、loadEnv 环境变量

---

## 为什么需要环境变量

开发：

```bash
http://localhost:8080
```

测试：

```bash
https://test-api.xxx.com
```

生产：

```bash
https://api.xxx.com
```

不能写死。

---

## Vite 的环境文件

---

### .env.development

```env
VITE_API_URL=http://localhost:8080
```

---

### .env.production

```env
VITE_API_URL=https://api.xxx.com
```

---

## 读取

```ts
const env = loadEnv(mode, process.cwd())
```

---

## 使用

```ts
env.VITE_API_URL
```

---

## 为什么必须 VITE_ 开头

Vite 安全机制。

只有：

```env
VITE_XXX
```

才允许暴露给前端。

否则：

数据库密码可能泄漏。

---

# 八、css 配置

---

## 全局 SCSS 变量

```ts
css: {
  preprocessorOptions: {
    scss: {
      additionalData: `
        @import "@/styles/variables.scss";
      `
    }
  }
}
```

作用：

所有 vue 文件自动拥有：

```scss
$primary-color
```

不用每个文件 import。

---

# 九、build 打包配置

这是生产环境核心。

---

## outDir

```ts
outDir: 'dist'
```

最终输出目录。

---

## sourcemap

```ts
sourcemap: false
```

生产是否暴露源码映射。

开启后：

浏览器能看到源码。

很多公司生产关闭。

---

## minify

```ts
minify: 'esbuild'
```

压缩代码。

---

# 十、rollupOptions（高级核心）

Vite 底层打包器：

> Rollup

---

## manualChunks

```ts
manualChunks: {
  vue: ['vue']
}
```

意思：

把 vue 单独拆包。

---

## 为什么拆包

否则：

一个巨大 bundle。

首屏慢。

---

## 拆包后

浏览器缓存：

```bash
vue.xxx.js
```

以后不用重复下载。

---

# 十一、define 全局常量注入

```ts
define: {
  __APP_VERSION__: JSON.stringify('1.0.0')
}
```

于是代码里：

```ts
console.log(__APP_VERSION__)
```

直接能用。

---

# 十二、Vite 为什么比 Webpack 快

核心思想完全不同。

---

## Webpack

启动：

```text
先打包整个项目
再启动服务器
```

大型项目：

几十秒。

---

## Vite

利用浏览器原生 ESModule：

```text
按需编译
访问哪个文件编译哪个
```

所以：

秒开。

---

# 十三、真实企业项目里常见配置

---

## 1）自动导入

```ts
unplugin-auto-import
```

---

## 2）组件自动注册

```ts
unplugin-vue-components
```

---

## 3）gzip压缩

```ts
viteCompression()
```

减少体积。

---

## 4）CDN 外链

生产环境：

```ts
vue 不打进 bundle
```

改为：

```html
<script src="cdn"></script>
```

进一步减小体积。

---

## 5）Mock接口

```ts
vite-plugin-mock
```

前端独立开发。

---

## 6）权限路由

动态生成菜单。

---

# 十四、一个现代 Vue 项目真正的启动流程

实际上：

```text
npm run dev
```

背后发生：

---

## 第一步

Node 执行：

```bash
vite
```

---

## 第二步

读取：

```bash
vite.config.ts
```

---

## 第三步

加载插件系统。

---

## 第四步

创建开发服务器。

---

## 第五步

浏览器访问：

```bash
localhost:5173
```

---

## 第六步

浏览器请求：

```ts
main.ts
```

---

## 第七步

Vite 实时编译：

```vue
.vue → js
scss → css
ts → js
```

---

## 第八步

通过 ESM 返回浏览器。

---

# 十五、为什么很多人看不懂 vite.config

因为它本质上同时涉及：

| 领域       | 内容         |
| -------- | ---------- |
| Node.js  | 配置运行环境     |
| Rollup   | 打包原理       |
| ESModule | 模块系统       |
| 浏览器      | dev server |
| 网络       | proxy/CORS |
| 工程化      | 环境变量       |
| 编译原理     | 插件转换       |
| 前端性能     | chunk拆分    |

所以：

> vite.config 本质是“前端工程化总入口”。

不是简单配置文件。

---

# 十六、建议的学习顺序（非常重要）

不要一上来全学。

推荐顺序：

---

## 第一阶段（必须）

先懂：

* alias
* server
* proxy
* env
* build

够用了。

---

## 第二阶段

再学：

* plugins
* 自动导入
* 组件自动注册

---

## 第三阶段（进阶）

再深入：

* Rollup
* chunk拆分
* CDN
* tree shaking
* 动态导入
* SSR

---

# 十七、如果继续深入，下一步应该学什么

建议你下一步深入：

1. `package.json`
2. `tsconfig.json`
3. `eslint.config.js`
4. `.env`
5. `src/main.ts`
6. `router`
7. `pinia`
8. `axios封装`
9. `vite插件开发`
10. `Rollup原理`

这些加起来：

才是真正的“现代前端工程化”。
