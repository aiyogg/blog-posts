最近在博客文章页面加上了一个阅读进度条，功能实现起来很简单，监听`scroll`事件改bar的宽度，点击bar滚动到相应位置。
但是这期间遇到一个小问题，小屏下，原来的菜单栏覆盖了我新加的进度条。  
我一看，两个都是`fixed`定位，凭直觉应该是我的进度条`z-index`设置不够大。自信加大`z-index`之后，发现
并没有变成我想要效果。
经过我一系列的翻查资料并用代码还原场景测试，基本确定这是`Chrome`浏览器的`bug`。
不过这是后事，这篇主要记录总结与此相关的知识点。

### 从问题开始
以下代码能基本还原我遇到的问题  
HTML代码：
```html
<body>
  <div id="div1"></div>
  <div id="div2" class="fadeIn">
    <div id="div2-1"></div>
  </div>
</body>
```
CSS代码：
```css
#div1 {
  width: 50px;
  height: 200px;
  background-color: red;
  position: fixed;
  z-index: 1;
}
#div2 {
  width: 200px;
  height: 200px;
  background-color: green;
  position: relative;
}
#div2-1 {
  width: 200px;
  height: 50px;
  background-color: yellow;
  position: fixed;
  z-index: 2;
}
.fadeIn {
  animation-duration: 2s;      
  animation-name: fadeIn;
}
@keyframes fadeIn {
  0% {
    opacity: 0
  }
  to {
    opacity: 1
  }
}
```
我博客中现象为红色`#div1`覆盖在黄色`#div2-1`之上，而上面的`demo`最终效果相反。

### 层叠上下文
由现象看本质，先来梳理相关知识点。
1. 概念  
  我们将一张网页看做是一个三维结构，页面中的元素延屏幕向用户（z轴）的一个立体空间上堆叠，
  如果一个元素含有层叠上下文，我们可以理解为这个元素在z轴上就“高人一等”。
      > 与此相关的还有一个抽象概念：**层叠水平**。层叠水平（stacking level）决定了同一个层叠上下文中元素在z轴上的显示顺序。  
      有一个特性是： 普通元素的层叠水平优先由层叠上下文决定，因此，层叠水平的比较只有在**当前**层叠上下文元素中才有意义。
2. 层叠上下文特性
   - 层叠上下文的层叠水平要比普通元素高；
   - 层叠上下文可以阻断元素的混合模式(`mix-blend-mode`)；
   - 层叠上下文可以**嵌套**，内部层叠上下文及其所有子元素均受制于外部的层叠上下文。
   - 每个层叠上下文和兄弟元素独立，也就是当进行层叠变化或渲染的时候，只需要考虑后代元素。
   - 每个层叠上下文是自成体系的，当元素发生层叠的时候，整个元素被认为是在**父**层叠上下文的层叠顺序中。
3. 层叠上下文的创建(重点)
    - 页面根元素天生具有层叠上下文，称之为“根层叠上下文”
    - z-index值为数值的**定位**元素的传统层叠上下文(必须是定位：`position`不为`auto`)
    - 其他CSS3属性
      - `z-index`值不为`auto`的`flex`项(父元素`display:flex|inline-flex`).
      - 元素的`opacity`值不是1.
      - 元素的`transform`值不是`none`.
      - 元素`mix-blend-mode`值不是`normal`.
      - 元素的`filter`值不是`none`.
      - 元素的`isolation`值是`isolate`.
      - `will-change`指定的属性值为上面任意一个。
      - 元素的`-webkit-overflow-scrollin`g设为`touch`.


### 层叠顺序
  1. 概念
    层叠顺序表示元素发生层叠时候有着特定的垂直显示顺序规则。  
    在CSS2.1的年代，在CSS3还没有出现的时候（注意这里的前提），层叠顺序规则遵循下面这张图：
    ![层叠顺序规则](http://olxg48efw.bkt.clouddn.com/article_img/stacking-order.png)
  2. 层叠准则
      1. 谁大谁上：当具有明显的层叠水平标示的时候，如识别的z-indx值，在同一个层叠上下文领域，层叠水平值大的那一个覆盖小的那一个。通俗讲就是官大的压死官小的。
      2. 后来居上：当元素的层叠水平一致、层叠顺序相同的时候，在DOM流中处于后面的元素会覆盖前面的元素。

### 回到问题上
根据以上关于层叠上下文、层叠顺序的知识，大概可以看出问题所在。  
`#div1`是`z-index`值为数值的元素，所以有独立的层叠上下文，`#div2-1`也是如此。
关键是`#div2`！它是定位元素(`position: relative`)，但它没有数值`z-index`，所以它没有创建独立的层叠上下文...吗？  
如果`#div2`并没有独立的层叠上下文，那么`#div1`和`#div2-1`处于同一父层叠上下文(根层叠上下文`html`元素)中
遵循“谁大谁上”的层叠准则那么`#div2-1`应该应该在`#div1`之上显示。
实际上并没有这么简单。`#div2`上有一个`fadeIn`的过度动画。`fadeIn`过渡动画是通过改变元素的`opacity`属性来实现。
所以动画执行完之前，`#div2`通过“不为0的`opacity`”这种方式创建了执行上下文。这期间`#div1`与`#div2-1`不再处于同一个父层叠上下文。
根据层叠上下文“嵌套”的特性，此时的`#div2-1`的层叠水平受限于其父层叠上下文`#div2`，`#div1`与`#div2`处同一级，而`#div1`层叠水平大于`#div1`
（根据层叠规则），因此`#div1`显示在`#div2-1`之上，等待过渡动画执行完`#div2`再次失去独立的层叠上下文，从而恢复之前的状态。  
观察最开始的demo也是这样这样的实际效果。但是我博客中的实际代码却不这样，左侧的navbar一直在进度条之上，而当我打开
控制台，手动移除进度条容器元素上的`fadeIn`类时才正常显示。因此我认为这是一个bug。  

### 总结
回顾整个过程，我从一开始认定是`z-index`的值大小导致的问题，修改之后发现并没有生效。这就暴露了自己最相关知识点的不熟悉，只是知道皮毛
不知其所以然。经过一系列的验证与探究，发现平时经常用的一条`z-index`背后竟然有如此多的规则细节，让我感叹就算是最基本的`CSS`知识
我还是存在欠缺。以后的工作与学习过程中，一定要多实践，多钻研，扫除知识盲区，特别是经常用到的，并多记录总结，以提高自己专业水准。

### 参考资料
1. [深入理解CSS中的层叠上下文和层叠顺序](http://www.zhangxinxu.com/wordpress/2016/01/understand-css-stacking-context-order-z-index/)
2. [z-index层叠上下文](http://www.jianshu.com/p/d5cc3e1e432c)
3. [层叠上下文 Stacking Context](http://www.cnblogs.com/elcarim5efil/p/4764607.html)