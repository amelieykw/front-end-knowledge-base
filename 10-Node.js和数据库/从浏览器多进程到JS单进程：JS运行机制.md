# [从浏览器多进程到JS单进程，JS运行机制最全面的一次梳理]()

## Summary
- 进程 VS 线程
- 浏览器进程
- 浏览器内核运行
- JS引擎单线程
- JS事件循环机制
--------
1. 区分进程和线程
2. 浏览器是多进程的
    - 浏览器都包含哪些进程？
    - 浏览器多进程的优势
    - 重点是浏览器内核（渲染进程）
    - Browser进程和浏览器内核（Renderer进程）的通信过程
3. 梳理浏览器内核中线程之间的关系
    - GUI渲染线程与JS引擎线程互斥
    - JS阻塞页面加载
    - WebWorker，JS的多线程？
    - WebWorker与SharedWorker
4. 简单梳理下浏览器渲染流程
    - load事件与DOMContentLoaded事件的先后
    - css加载是否会阻塞dom树渲染？
    - 普通图层和复合图层
5. 从Event Loop谈JS的运行机制
    - 事件循环机制进一步补充
    - 单独说说定时器
    - setTimeout而不是setInterval
6. 事件循环进阶：macrotask与microtask
------

## 1. 区分进程和线程

```
- 进程是一个工厂，工厂有它独立的资源

- 工厂之间相互独立 -> 进程之间相互独立

- 线程是工厂中的工人，多个工人协作完成任务 -> 多个线程在进程中协作完成任务

- 工厂内有一个或多个工人 -> 一个进程由一个或多个线程组成

- 工人之间共享空间 -> 同一进程下的各个线程之间共享程序的内存空间（包括代码段、数据集、堆等）
```

- **进程**是cpu资源分配的最小单位（是能拥有资源和独立运行的最小单位）
- **线程**是cpu调度的最小单位（线程是建立在进程的基础上的一次程序运行单位，一个进程中可以有多个线程）

- 不同进程之间也可以通信，不过代价较大
- 现在，一般通用叫法：**单线程和多线程**，都是指在**一个线程内**的单和多。（所以核心还是得属于一个进程才行）

----

## 2. 浏览器是多进程的

- 浏览器是多进程的
- 浏览器之所以能够运行，是因为系统给它的进程分配了资源（cpu，内存）
- 每打开一个Tab页，就相当于创建了一个独立的浏览器进程：
    - 浏览器应该也有自己的优化机制，有时候打开多个Tab页后，可以在Chrome任务管理器中看到，有些进程被合并了，例如打开多个空白页（所以每一个Tab标签对应一个进程并不一定是绝对的）

#### 浏览器多进程的优势

- 避免单个page crash影响整个浏览器
- 避免第三方插件crash影响整个浏览器
- 多进程充分利用多核优势
- 方便使用沙盒模型隔离插件等进程，提高浏览器稳定性

**当然，内存等资源消耗也会更大，有点空间换时间的意思。**

### 浏览器内核

#### 浏览器都包含哪些进程？

1. **Browser进程**：浏览器的主进程（负责协调、主控），只有一个。
    - 作用：
        - 负责浏览器界面显示，与用户交互。如前进、后退等。
        - 负责各个页面的管理，创建和销毁其它进程
        - 将Renderer进程得到的内存中的Bitmap，绘制到用户界面上
        - 网络资源的管理，下载等
2. **第三方插件进程**：每种类型的插件对应一个进程，仅当使用该插件时才创建
3. **GPU进程**：最多一个，用于3D绘制等
4. **浏览器渲染进程（浏览器内核）（Renderer进程，内部是多线程的）**：默认每个Tab页面一个进程，互不影响。
    - 作用：
        - 页面渲染，脚本执行，事件处理等

#### 浏览器渲染进程都包含哪些线程？

![浏览器渲染进程都包含哪些线程？](https://user-gold-cdn.xitu.io/2018/1/21/1611938b2d39a5b2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)


> 浏览器渲染进程 = 解析HTML和CSS文件，加载图片等资源文件，渲染成用户看到的页面 (**GUI渲染线程**) + 执行解析js文件脚本代码 (**JS引擎线程**)

1. **GUI渲染线程**

![GUI渲染线程](./图片/浏览器GUI渲染进程.jpg)

我们主要分析GUI渲染线程执行的详细过程：

1. 解析HTML文件，构建DOM树，同时浏览器主进程负责下载CSS文件。
2. CSS文件下载完成，解析CSS文件成树形的数据结构， 然后结合DOM树合并成RenderObject树。
3. 布局RenderObject树，负责RenderObject树中的元素的尺寸，位置等计算。
4. 绘制RenderObject树，绘制页面的像素信息。
5. 浏览器主进程将默认的图层和复合图层交给GPU进程，GPU进程再将各个图层合成(composite)，最后显示出页面。
    - 默认图层：指出于普通文档流的元素
    - 复合图层：指使用动画执行或者``<video><iframe><canvas><webgl>``等元素，也可以使用z-index将层级高的元素变成复合图层，使用复合图层可以进行硬件加速，其原理是避免了默认图层的重绘和回流。


```
了解GUI渲染线程的执行过程，我们可以根据原理进行渲染优化：

1. 尽可能早的提前引入CSS文件
    - 例如在头部引入CSS文件

2. 尽可能早的加载CSS文件中的引入的资源，可以使用预加载，在link标签中加入rel="preload" as="font"该元素属性，不会造成渲染阻塞。
    - 例如自定义字体文件

3. 在DOM和CSS渲染之后加载js文件
    - 例如在尾部加载js文件，或者使用该元素属性defer和async，进行js问价异步加载，但是不同浏览器会有兼容性问题。
```

当界面需要重绘（Repaint）或由于某种操作引发回流（reflow）时，该线程就会执行。

注意，GUI渲染线程与JS引擎线程是互斥的，当JS引擎执行时GUI线程会被挂起（相当于被冻结了），GUI更新会被保存在一个队列中等到JS引擎空闲时立即被执行。

[TODO]

[浏览器的运行机制—3.浏览器的渲染进程](https://www.jianshu.com/p/05606b0b4eb1)

[DOM 是什么?](https://www.zhihu.com/question/34219998)

[渲染机制/页面性能/错误监控](https://segmentfault.com/a/1190000018342166)

[(全网方案整合商)从url输入到建立自己的前端知识体系(巨图警告)](https://segmentfault.com/a/1190000021939250)


2. **JS引擎线程**


3. **事件触发线程**


4. **定时触发器线程**

5. **异步http请求线程**