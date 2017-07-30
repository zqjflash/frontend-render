## 前端渲染优化

### 一、目录

1、浏览器渲染原理

2、如何优化分层

3、网页动画

4、H5 Canvas的优化

### 二、页面是如何显示出来的？

![Alt text](/2-1.png)

先简单总体说下浏览器是如何显示一个页面的，要经过哪些步骤？
一个网页要显示出来需要经过加载、解析、排版和渲染几个过程。

加载：就是指从网上把页面资源都拉过来；

解析：就是分析html文件生成dom树；

排版：是开始计算各个节点的显示的位置坐标；

渲染：就是将dom树中各个节点都画出来最终呈现出网页的可视画面。之前跟一些前端同学打过交道，可能很多同学口中的网页渲染说的是页面元素该显示在什么位置或者什么样式。这其实大部分说的是页面排版的过程。

![Alt text](/2-2.png)

那么网页真正渲染的过程是这样的。主要涉及下面几个线程：render线程、合成线程以及光栅化线程。

不知道前端同学对线程是否了解。线程可以理解为：每个线程都是独立的执行任务，相互之间是异步的，能同时并发执行各自的任务。在来看这张图，renderer线程是内核运行的主线程，其中解析排版以及js执行都在这个线程，也只有这个线程能访问和操作dom树，所以只要跟dom树沾边的操作或任务就只能在这个线程中执行。

合成线程主要工作就是渲染显示。光栅化线程唯一任务就是光栅化了。

接着看流程：排版完成后页面各个元素的位置样式等都确定了。

接着开始录制。录制就是把页面的元素变成一条条绘制的指令。比如在xxx位置画xx图片，在xx位置显示一行文字等等。

录制完成后会将录制好的内容交给合成线程。合成线程会先把整个页面进行分块，一个页面分成若干个小的分块。在逐个将每个分块放到光栅化线程执行光栅化。

光栅化其实就是把之前录制好的指令真正地画成图像。光栅化完成后就形成了一个个分成块的网页的图像。

接着在合成线程根据当前屏幕大小和页面滑动的位置等，选择好要显示的分块，再把这些分块一个个滑到屏幕上。这样我们就看到了网页的显示画面了。

#### 2.1 网页分块渲染

![Alt text](/2-3.png)

前面将页面显示的时候要进行分块。那么为啥要分块呢？

这主要是为了提升操作响应速度。网页都分块了，那么无论如何滑动或者缩放页面，浏览器都能很快地挑出需要显示的分块在合成画到屏幕上。
分块还有一个好处就是可以按照优先级来绘制页面各部分的内容。比如马上需要显示的部分肯定需要最先画出来。

看右侧这张图，浏览器会cache两种分辨率的分块，一个正常分辨率（像右图中的小字编号的分块），一个低分辨率的分代。这主要是为了防止来不及画，低分辨率画地快，有个模糊的画面比显示白色要好很多。

手机浏览器上，猛然滑动页面有时候会看到文字模糊的画面，这就是看到低分辨率分块了。
低分辨率是应急准备的，那么它的优先级肯定比高分辨率高，需要先通过光栅化画出来。

每个分辨率都是先画马上要显示的分块，也就是可见区域的分块。然后离可见区域越近的优先级越高，越早画出来。其实就是从可见的部分一圈圈往外画分块。

#### 2.2 网页分块渲染规则

为了滑动显示效果每个分层会保留至少两种分辨率的分块；

所有分块会根据到可视区域的距离来确定渲染的优先级；

硬件加速分块的载体是OpenGL Texture。

总体一下，页面是分块渲染的。一般会保存两种分辨率的分块。分块是按照优先级来决定谁先画。
现在浏览器都采用硬件加速渲染了。所以分块的载体是opengl的纹理。生成分块纹理的过程叫光栅化（Raster）。有两种另一种是GPU光栅化，就是指分块直接用GPU画的。

#### 2.3 页面分块的GPU光栅化

GPU光栅化速度更快，内存，CPU消耗较少；
目前还无法对包含复杂矢量绘制的页面进行GPU光栅化；
GPU光栅化是未来趋势。

GPU光栅化速度更快，消耗也较少。但是它有个很大的缺点，复杂矢量绘制效率很低，无法采用。
但是GPU光栅化是一个趋势。
对于前端来说，页面中少一些复杂线条是能够提升渲染的光栅化效率的。原来不能用GPU raster的页面，减少复杂线条，就能用GPU raster了。对于能够用GPU raster的页面，减少复杂线条能够提升raster速度。

### 三、网页的分层渲染

![Alt text](/3-1.png)

前面讲了页面是分成块进行渲染的，这里再讲下页面分层渲染。下面这幅图是用chrome浏览器看到的手机版搜狐页面的分层情况。

![Alt text](/3-2.png)

Dom树大家应该都比较熟悉了。这幅图估计也都见过很多遍了，网上一搜到处都是。Dom树排版以后生成了render树。最后的这个graphics layer树基本上就是对应显示的分层。

#### 3.1 分层渲染的优点

最主要的是能够减少不必要的重新绘制，一个层上的变化不会波及周边其它层；
可以实现较复杂加速动画，很多动画分层以后更容易实现；
能够方便实现复杂的CSS样式

#### 3.2 前端控制页面分层如何影响渲染效率？


对于页面分层渲染，前端应该要注意哪些呢？或者是前端应该如何控制页面分层而达到最优的渲染效率呢？

下面通过几个有问题的例子讲起

小豆腐块分层较多。页面整体分层总数量较大。会导致每帧渲染时遍历分层和计算分层位置耗时较长。

第一个例子，腾讯网。

![Alt text](/3-3.png)

腾讯网在android手机上显示的分层情况。可以看到很多新闻条目都被单独分层，像小豆腐块一样。

分层面积虽然不大，但是数量可观。对渲染性能会造成一定的影响。浏览器每帧都会遍历计算所有分层坐标等，还要算分块优先级啥的。分层多了计算量就变大了。
当然现在的一般android手机显示qq.com也没太多问题，影响不至于太大。但是低端手机上性能的影响还是能够体现出来的。这样毕竟会导致低效的渲染。
为啥会造成这样的结果呢。我们来看页面的代码。

上面图片就是这个“galleryinner”是可以左右滑动的，所以设置了transfrom属性。这样浏览器会进行分层。而后面的很多元素设置的position属性值为relative导致也被分层了。

所以就产生很多小块分层。后面这些小分层分层的原因用最新的chrome浏览器可以看到：是它们可能覆盖在一个已经分层的元素上，就是可能覆盖在这个galleryinnner。

第二个例子：可视区域内分层太多且需要绘制面积太大，渲染性能非常差。甚至达到无法正确显示的地步。

![Alt text](/3-4.png)

这个例子类似的页面和模板很常见。一些招聘，广告等都采用这种方式。但是有的这种模板是有渲染问题的，比如这个页面。
所有的页面都在可见区域内。当一层不显示了，就把它旋转90度竖着放在下面。是通过这几行代码实现的。
而浏览器需要渲染和处理可见区域内所有的分层。这个页面的分层总面积非常大，渲染会非常慢。另外，需要非常多的内存来cache渲染内容。
在一些手机上已经超过浏览器的内存使用限制，造成一些地方显示不出来。这个页面很多手机浏览器上都会出现闪烁。

第三个例子：人民网首页

前面举了两个分层多影响渲染性能的例子。那么是不是分层越少越好呢。
我们来看看人民网首页。这个页面比较极端，就一个分层。

![Alt text](/3-5.png)

当页面发生变化时，可以看到绿色框子中的重绘区域。类似的元素和图片左右滑动的效果，在各种手机页面中非常常见。
它们都是元素位置变化，而内容是完全不变的。这种情况如果不分层的话，浏览器每帧都要重绘所有变化的元素。动画的帧率会明显下降。

渲染角度分析：没有分层。页面变化时需要重绘的区域较多。元素内容无变化只有位置发生变化时，可以利用分层来避免重绘。

第四个例子：1号店首页

![Alt text](/3-6.png)

前面几个例子有分层多的，有完全不分层的。我们再来看最后一个例子，一号店的android版首页。
这个页面看起来分层还比较合适。该分层的都分层了。也没有太多分层。
但是~~~~~~~~
这个页面每隔一两秒，分层就发生一次变化。而且变化很大。原本在最底层的很多内容，被单独分成一个层了。
这样底层内容发生变化，需要完全重新绘制。新分出来的层肯定也是要完全重新画的。需要重新绘制的面积非常非常大。
假如像人民网一样完全不分层的话，重绘的面积还没有这么大。这就明显是弄巧成拙了。
为什么会这样呢，看代码。在这里有一个上下滚动的字幕。每隔一段时间会给它设置个transform属性和动画。动画完了再去掉。
这样的话这个元素平常是不分层的，而动画的时候立马进行了分层。动画完了再变回去。
后面其他元素的分层也是设置了position：relative属性引起的。在这里分层变化时，后面一堆元素的分层也跟着变化了。
这样重绘面积非常大，性能会比较差，也很耗电。

渲染角度分析：分层发生大面积变化，由此产生全屏重绘。还不如不分层渲染性能好。

#### 3.3 分层原因

每个浏览器或者不同版本的浏览器分层策略都会不同。但总体应该差不太多。
最常见的几个分层原因：transform、Z-index；硬件加速的video、canvas等；fixed的元素；还有一个常见的容易被忽视的分层原因是下层的元素被分层了。

#### 3.4 Chrome查看分层的工具

![Alt text](/3-7.png)

前面ppt中贴了很多能看分层的图。其实都是用chrome浏览器中的工具看的。
最常用，最稳定的办法是看分层的边缘网格线，这里面也包含分块的网格线。比如我们做内核渲染相关的工作，常年默认都是开着网格线。一眼就能够看出页面分层分块情况。
看网格线还有一个地方在这里，但是有的chrome版本貌似看不了。
还有前面直观地看出3D形状的分层效果图的是在chrome浏览器的设置放在一起 more tools下的layers。不同版本chrome可能位置不一样，我这个是M59的chrome。
这个工具很高大上，其实是chrome实验室功能。不太好用，非常卡很耗内存。
如何控制分层，因为分层的逻辑确实非常复杂，所以最靠谱的方式还是通过工具看分层。再尝试调整。

#### 3.5 到底如何分层才是渲染最优的呢？

分层目的主要是减少重绘，当元素有位置变化时适合分层。
既然浏览器分层的目的主要是减少重绘。那么当元素位置可能产生变化的时候就需要分层。
像下面几个页面的位置不断变化的图片。

![Alt text](/3-8.png)

#### 3.6 静止的元素周围的内容频繁变化的元素需要单独分层。如下图红框中的时间显示。

![Alt text](/3-9.png)

静止的元素周围内容频繁变化的元素需要单独分层。如下面淘宝网首页的这个时间显示。
时间周围的元素基本都是不动的。如果不分层的话，时间每隔1S变一下。浏览器不光要绘制时间这个元素，它底下的元素，还有周边的元素都可能需要重新绘制。
而分层了，就只需要绘制时间这一小块区域了。
淘宝这里确实也是分层了，是用了z-index属性分层的。

#### 3.7 虽然分层能够减少浏览器的重绘。但是物极必反，分层太多则会导致浏览器遍历和计算的耗时变长，分层面积过大会导致渲染所用内存增大。这些都是会影响渲染性能的。

下面这两个图就是刚才说的有问题的例子。他们的问题都是分层太多了，或者面积太大了。
分层只要能够达到减少重绘的目的就好，要防止层爆炸。

![Alt text](/3-10.png)

#### 3.8 哪种方式是渲染较优的？

![Alt text](/3-11.png)

![Alt text](/3-12.png)

红框中图片分层处理方式有两种：

* 所有图片一个长条分层，图片变换时显示长条分层的不同区间。
* 每个图片分一层，不显示的挪到视野左边或右边，显示时挪进来。

第二种是比较优的。原因是：
* 第一种分层个数较多，计算量会比较大。
* 内核会优先绘制离可见区域较近的内容。第一种未显示的分层都在可见区域周围，内核无法判断其渲染优先级。而第二种则可以根据离可见区域的距离来合理安排渲染和cache。

#### 3.9 总结

分层总的原则是：减少渲染重绘面积与减少分层个数和分层总面积。

* 元素只有相对位置变化的需要分层。
* 元素更新比较频繁需要分层。
* 较长较大的页面注意总的分层个数。
* 避免某一块区域分层过多，面积过大。

### 四、Animation动画

#### 4.1 介绍


前端：CSS 动画 or JS 动画
内核：Renderer线程动画 or Compositor线程动画

讲完网页的分块和分层，我们开始讲网页动画。提到动画，估计前端同学马上就会想到动画的实现方式是CSS动画呢还是js动画。
但是从浏览器内核渲染的角度，我们更关心这个动画是从renderer线程（也就是主线程）发起的还是合成线程发起的。
主线程或者叫renderer线程负责解析排版以及执行JS，是一个非常繁忙的线程，响应速度会比较低。所以必须要经过renderer线程处理的动画，就比较容易响应不及时造成卡顿等效率问题
而合成线程是负责上屏显示的，刷屏时一般都要求帧率达到60帧。所以它的响应速度是非常快的。所以完全由合成线程控制的动画则效率会非常高。

#### 4.2 js动画与CSS动画线程对比

![Alt text](/4-1.png)

这张图表示的是Renderer线程动画和合成线程动画浏览器处理的流程。不同的颜色代表该任务是在不同的线程中执行的。
Renderer线程动画，浏览器的处理流程是先处理动画逻辑，然后再经过排版和录制，以及光栅化再到合成上屏才能显示出来。当然这些过程并不是每个renderer线程的动画都必须执行的。比如JS只改变了分层的位置，并不会引发重新绘制，那么这个录制和光栅化就会空跑一下，并不会占用太多时间。
合成线程的动画流程就特别短了，执行动画的逻辑后就合成上屏了。而且都在合成线程中，响应速度很快。
那么哪些动画是renderer线程动画，哪些动画是合成线程的呢？
JS动画，因为JS是必须要在renderer线程执行的，所以它肯定是renderer线程动画。
CSS动画有一部分是需要经过renderer线程处理的。有一些则可以完全在合成线程中处理。

#### 4.3 不会触发重绘或排版的CSS动画属性

cursor
font-variant
opacity
orphans
perspective
perspective-origin
pointer-events
transform
transform-style
widows

这里给出一些不会触发重绘和排版，完全可以在合成线程中处理的CSS动画的属性。
其中我们比较常用的可能就是opacity透明度，以及transform的动画。后面这个二维码对应的页面中有一个比较详细的列表可以参考。
既然合成线程的动画效率会比较高，那么也就是说上面列的这些都是比较高效的动画方式。

#### 4.4 Renderer线程动画一定比Compositor线程动画的性能差么？

前面讲compositor线程要比renderer线程的动画好。可能有些同学表示比较怀疑。我曾经两种方式都用过，没发现性能有多大差别啊。
Renderer线程的动画流程比较长，经过的线程比较多。性能肯定是会差一些。如果动画没有触发严重的排版和重绘的话，在PC或者性能好一点的移动终端上，多出来的耗时也不会造成FPS的明显差异。
但是在低端设备上，这个耗时造成的影响就会比较大了。

![Alt text](/4-2.png)

#### 4.5 重排版和重绘对renderer线程动画的影响

![Alt text](/4-3.png)

重排和重绘对renderer线程动画的影响。正常的动画时序是这样的。
当有重排和重绘时，会将合成上屏的逻辑延后了，也就是产生掉帧了，这里就掉了一帧。

#### 4.6 重排版和重绘对compositor线程动画的影响

![Alt text](/4-4.png)

对于合成线程的动画，重排和重绘同样也可能造成掉帧。这个合成上屏的逻辑被推迟了。但是，这个影响要比前面的js动画要好的多。

#### 4.7 右图中所有的心型和三角形的动画都是CSS Animation。

* Animation 动画的元素很多，导致分层个数非常多。浏览器每帧都需要遍历计算所有分层，比较耗时。

![Alt text](/4-5.png)

右边这种页面大家肯定见过类似的。图中的心形和三角形都是css动画。那么从渲染的角度来看，这个页面可能存在哪些渲染的问题呢。
主要的问题就是分层太多了。
浏览器内核每帧都会多次去遍历所有分层进行相关的计算。CSS3动画的分层还是3D的，处理更复杂。 计算产生的耗时就会比较大。容易导致掉帧情况发生。

* 如何解决大量CSS Animation带来的渲染性能降低？

![Alt text](/4-6.png)

那么是不是有更好的办法来处理上述的问题呢。既然分层多了，是不是得减少分层呢。是不是可以进一步利用硬件加速呢。
那么用H5 2D的canvas作为替换方案是个比较好的选择。

* 2D Canvas相比CSS Animation优点

硬件加速渲染
      目前基本所有浏览器2D Canvas都是采用硬件加速渲染的。对于贴图操作GPU非常擅长。
渲染流程更优
      任务更容易拆解，渲染过程可以多线程异步执行。

![Alt text](/4-7.png)

左边的用例采用的是CSS animation，虽然就几十个心型和三角形，但是在中低端手机上帧率可能不会太高。
右边满屏的子弹飞机，在中低端手机上帧率也能达到50以上。两者的渲染效率差距还是比较大的。
其实现在的手机端web游戏绝大部分都是H5 2D或者 3Dcanvas做的。当然也有一小部分是CSS实现的。我们分析过，从渲染角度看CSS的游戏性能确实是差很多的。

* H5 2D Canvas渲染流程

![Alt text](/4-8.png)

前面讲2D canvas的渲染流程更优，那么我们就来看下它的渲染流程为什么更优。
2D canvas的绘制都是JS调用的。JS肯定是在主线程也就是renderer线程。需要注意的是JS调用绘制的时候其实并没有真正的去画。这里其实也是记录绘制的操作。之前见过前端同学在这里用js来统计渲染耗时啥的，其实是不准确的。录制完成后会进行回放，但是真正执行渲染的opengl指令是在另一个GPU线程去执行的。最后上屏还是在合成线程。
这个回放的操作在一些浏览器中其实是分散到别的线程中的。

也就是说canvas绘制的任务被拆分成几个部分，分散到几个线程中并发执行。浏览器都对其进行了针对性的优化，所以它的帧率会比普通元素要好。

* HTML 2D Canvas三种主要绘制元素：

从内核角度来看，就三种主要的绘制元素：图片，文字，和线条矢量。

* 硬件加速图片的绘制

![Alt text](/4-9.png)

图片的绘制，第一步肯定是解码了。解码后的图片会存在内存中。Canvas是完全硬件加速的使用gpu绘制的。所以要将图片上传到GPU的纹理中，形成一个纹理的cache。最后再用GPU贴图。

* 善用图片Cache

![Alt text](/4-10.png)

对于这个纹理的cache，前端同学需要注意几个方面。
将多个小图片资源拼接成一个大图片，也就是雪碧图。为什么建议这样做呢？因为GPU是流水线式工作的。一些状态的改变会导致性能的明显降低。图片都拼成一张雪碧图的话，那么基本上所有的图片绘制都是将A图片画到目标上，浏览器也比较容易进行批处理的优化。如果是一堆小图片的话，那么这次是A画到目标上，下次再画B，状态不停地切换，性能也就降低了。
其实这点大部分前端都是这样做的。不过，雪碧图的size也不宜过大，移动终端上建议不超过4096.这个原因是因为GPU的纹理是有最大size限制的，太大了会创建不出来。另外大内存块的申请失败率会比较高。对于超大的图片，浏览器会对其切割处理，比如一切四等等。这个切割肯定不会太合理，很容易把一张小图切成两部分。绘制效率也就降低了。
图片size为2的幂时cache性能最好。这主要因为opengl或者GPU中纹理存储都是基于2的幂方式组织的，早先的GPU甚至都不支持非2的幂的纹理。不过这点也不需要刻意强求。现在的GPU上两者差距也不会太大。
图片资源总量如果较大时需要自己创建cache。这是因为图片cache的总大小是有限的。如果资源总量超过这个限制，就会存在频繁进出cache的情况。性能会大大降低。这是就需要自己创建cache。创建的方法，可以用两个canvas来当做一张cache图片。

* 硬件加速文字的绘制

![Alt text](/4-11.png)

文字的绘制其实也是先建立cache。根据字体，文字的高度和文字编码来建立cache。然后再绘制。

* 文字Cache的几种形态

![Alt text](/4-12.png)

相比图片，文字的cache是要复杂的多。
第一个cache是glyph cache。这个是文字编码到所属字体的glyph index编码的查找表。Glyph index可以理解为，比如雅黑中的这个“到”字在雅黑字体文件中保存的位置索引。
文字宽度查找表，这个是排版时用到的。文字显示占用的宽度其实是不一样的，比如这个W和i。排版的时候需要知道字的宽度，而每次从字库中取会比较耗时，那么就会建立一个cache。
这个bitmap的cache保存的是单个字符指定高度的点阵图。现在基本都是矢量字体，而最终显示需要渲染成点阵图。这个事情是字体引擎干的，这个过程是相对比较耗时的。可以说一个字的渲染大部分时间都耗在这里。
硬件加速的H5 canvas等会把点阵图上传到纹理中。这个上传纹理一般也是比较耗时的。
总的来说，文字的cache的建立是非常耗时的。有cache和没cache的绘制时间要差一个数量级。也就是说文字的绘制时间基本都耗在建立cache上面。而cache是每个字不同大小都要重新建立，canvas中的字如果都是重复的相同的几个字，画起来还是挺快的。否则绘制也是比较耗时的。

* 硬件加速矢量的绘制

简单的矢量图形，比如点、直线等可直接使用OpenGL渲染。
复杂的矢量图形，如曲线等，无法采用OpenGL绘制
Canvas中的矢量绘制，简单的矢量，如点和直线等可以直接用opengl来渲染。而复杂的矢量图形，如曲线等就没法用opengl绘制。
总体来说，GPU并不擅长画矢量。所以canvas中的矢量线条等，绘制效率会相对比较差。

* 2D Canvas绘制效率

GPU或者opengl是批量处理的，最适合干的事情就是贴纹理。所以你看很多绘制文字，不能用opengl直接画的矢量，最后都变成帖纹理。
讲了这么多其实主要要讲得就是，canvas绘制图片比较擅长。画文字和线条是不擅长的。在使用canvas的时候，需要注意这一点。

* 2D Canvas性能瓶颈在哪里？

![Alt text](/4-13.png)

这里有个问题，前面将浏览器把2D canvas的任务拆解到几个线程并行处理。那么这几个任务中最繁重，最容易影响整个渲染效率的是哪个呢？
是JS的执行，还是这个回放，或者是执行opengl命令真正的渲染？

我们分析了很多canvas用例，发现canvas绘制中其实是JS最容易成为瓶颈。很多人可能以为渲染性能差，那不应该是真正执行渲染的这个opengl耗时了么。其实并不是。当然并不是每个canvas都是js瓶颈。如果文字或者矢量太多的话，那瓶颈估计就不一定是js 了。

* H5 2D canvas无法克服的问题

JS的单线程执行与CPU多核化发展趋势之间的冲突。
JS引擎的优化暂时还无法满足canvas性能要求。
有限的绘制接口无法满足日益增长的需求。

前面将 2D canvas中js执行是一大瓶颈。那么这个有办法解决么？
虽然浏览器把canvas绘制任务分散到其他线程了。但是canvas绘制的JS的执行必须和dom树操作在一个线程—也就是renderer线程，暂时无法分线程处理。
硬件设备的发展方向是多核化。单个cpu核的频率提升应该很小了。而单线程的JS执行慢恰恰需要提升cpu频率才行。也就是来说，就是硬件升级也无法缓解js执行慢的问题。

Js引擎的优化，可能会对2d canvas性能产生正面影响。但是js引擎出现质的变化会比较困难。

还有一点，canvas的绘制接口太少了。做不出比较炫酷或者3D的效果来。

#### 4.8 小结

尽量使用不会引起重绘的CSS属性动画，如transform、opacity等。
动画一定要避免触发大量元素重新排版或者大面积重绘。
在有动画执行时，避免其他动画不相关因素引起的排版和重绘。

总结一下，对于网页中的动画尽量使用合成线程的动画。也就是之前列出的一些不会引发重新排版和绘制的一些CSS动画。比如transform或者opacity等。
如果确实有需要使用其他类型的CSS动画或者JS动画的话，那么一定要避免触发大量元素的排版和大面积的重绘。比如改变了某个元素的样式或者位置，引发依赖它的其他元素的一系列排版变化。对于大面积重绘，前面讲过的那个一号店的例子就是动画引发了分层变化，导致了大面积的重绘。
还有一点需要注意的是，在有动画执行的时候。也要注意网页的其他部分逻辑引起的排版和重绘。因为renderer线程只有一个，如果有其他任务占用了，那么必然就没法很好的响应动画了。合成线程响应虽然快，但是大量的重绘仍然可能引发掉帧。

### 五、WebGL

