+++
title = "js屏幕缩放原理"
date = 2017-10-10T00:00:00+08:00
tags = ["javascript", "js"]
categories = ["javascript"]
draft = false
author = "7ym0n"
+++

{{< figure src="/javascript.jpg" >}}


## 实现对页面的预览 {#实现对页面的预览}

-   [X] 采用原生 js 实现页面缩放功能，需要计算<code>[100%]</code>：
    -   [X] 屏幕宽度与高度；
    -   [X] 缩放比例计算；
    -   [X] 宽度高度改变联动；


### 获取屏幕高宽度 {#获取屏幕高宽度}

获取默认的宽度与高度：

```javascript
if (window.innerHeight && window.innerWidth){
  this.height = window.innerHeight;
  this.width = window.innerWidth;
}else if ((document.body) && (document.body.clientHeight)){
  this.height = document.body.clientHeight;
  this.width = document.body.clientWidth;
}
```


### 计算缩放比例 {#计算缩放比例}

对需要缩放的内容进行计算，计算公式：\`容器高度/缩放对象高度 = scale\`。容器是缩放对象的参照物，假设需要缩放 body 里面的 div 矩阵（即缩放对象），那么容器即为 body 本身；
例如：

```html5
<div id="panel_container" style="width:1066.656px;height:600px; border: 1px solid;">
    <div id="screenshot" style="width:100%;height:100%;">
        <div id="preview" style="width:1920px;height:1080px; border: 1px solid;">

        </div>
    </div>
</div>
```

那么需要计算 preview 的缩放比例，就是：\`600/1080 = 0.55555\`,设置 previe 的 transform 样式 scale(0.55555);即得到了正确的缩放；


### 宽度高度改变联动 {#宽度高度改变联动}

   在计算缩放比例后，同时需要计算容器的宽度，需要采用缩放比例进行计算，计算公式：\`缩放对象的宽度\*缩放比例 = 容器宽度\`；即：\`
1920 \* 0.55555 = 1066.656 \`
