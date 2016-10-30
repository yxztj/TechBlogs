# AsyncDisplayKit介绍（二）布局系统

在上一篇介绍中我们曾经讨论过Autolayout的性能问题。然而在iOS中，除了Autolayout，能选择的只有autoresizingMask，或者纯手动布局。在写了无数view.frame = CGRect(…)之后，我们才发现，一个在HTML中非常简单的流式布局，到iOS9才有相应的UIStackView予以支持。尽管你可以使用[FDStackView](https://github.com/forkingdog/FDStackView)将系统要求降低到iOS6，对于复杂的布局仍然需要多层View嵌套来实现，增加了不必要的开销。

做过Web开发的同学们已经习惯了高效的css布局：声明式、易于调试，boxModel结构化清晰等等优点，特别是使用自动化工具之后只要单单保存文件就可以立即在浏览器中看到效果；而这一切在iOS中就变得遥不可及：命令式赋值、编译后（Objc速度尚可，而swift编译速度较慢）才能看到结果、Autolayout闭源、视图调试复杂、xcode还经常crash等等。如果要实现一个高性能的tableView，手写布局几乎是唯一选择。

Facebook将React的概念延伸到native，同时也把声明式的语法带到了iOS，产生了[ComponentKit](https://github.com/facebook/componentkit)框架。早在ASDK1.0的年代大家纷纷表示希望能将其与ComponentKit相融合，幸运的是，在2.0版本中实现了。我选用了作者Scott
Goodson在NSSpain的[演讲pdf](https://github-cloud.s3.amazonaws.com/assets/repositories/21265042/15788?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAISTNZFOVBIJMK3TQ/20160204/us-east-1/s3/aws4_request&X-Amz-Date=20160204T100616Z&X-Amz-Expires=300&X-Amz-Signature=50d624c30b9b31a059b1bf299a615bd72418c420e35d3730cc0017e0b3bfcf73&X-Amz-SignedHeaders=host&actor_id=3334458&response-content-disposition=attachment;filename=AsyncDisplayKit.2.0.Beta.-.October.2015.Slides.pdf&response-content-type=application/pdf)作为插图，方便大家了解新的布局系统。

## AsyncDisplayKit的三种布局方式

###1.  手动布局

我们知道，对于每一个待布局的UIView，它需要报告自己的大小(sizeThatFits)，同时对它的subviews进行布局(layoutSubviews)，如此层层递归构成一个完整的自上至下的布局。

同样，在ASDK中的手动布局也是类似的，只是方法名变成了calculatedSizeThatFits和更加直白的layout。略微不同的是ASDK会将布局结果缓存下来以提升性能，这点对于tableView滚动性能来说有非常大的帮助。

然而，跟普通手动布局类似，最大的缺点是可读性和可维护性差。由于自身的大小常常和子元素的布局相关联，这两个方法中的代码容易重复；同时由于布局针对自身subView，很难将其代码与其他View进行复用。

###2. Unified layout

这也是在ASDK中新出现的概念。先介绍一下ASLayout：

![](https://cdn-images-1.medium.com/max/1600/1*kSCV71Snhq1067mLrn7xwA.png)

一个ASLayout对象包含以下元素：

* 它所代表的布局元素
* 元素的尺寸
* 元素的位置
* 它所包含的sublayouts

可以看出，当一个node具备了确定的ASLayout对象时，它自身的布局也就随之确定了。为了生成ASLayout，ASDisplayNode为subclass提供了如下覆盖点：

    - (ASLayout *)calculateLayoutThatFits:(ASSizeRange)constrainedSize

只要Node能够计算出自己的ASLayout，父元素就可以完成对其的布局。这种方法将sizeThatFits和layoutSubviews结合在一起，一定程度上避免了相似代码的尴尬，但是计算上仍然是手动布局，不够简便。

###3. Automatic Layout(不是Autolayout)

这是ASDK最推荐也是最为高效的布局方式。它引入了ASLayoutSpec的概念，可以理解为一个抽象的容器（并不需要创建对应node或者view来装载子元素），只需要为它制定一系列的布局规则，它就能对其负责的子元素进行布局。

我们先来看一下它们之间的关系：

![](https://cdn-images-1.medium.com/max/1600/1*QXBbQ4z7f8ypJs_HQZ_u_A.png)

可以看到ASLayoutSpec和ASDisplayNode都实现了ASLayoutable接口，因此他们都具备生成ASLayout的能力，这样就能唯一确定自身的大小。

* 对于以上提到的Unified布局，当我们实现了calculateLayoutThatFits，ASDK会在布局过程中调用measureWithSizeRange(如果没有缓存过再调用calculateLayoutThatFits)来算出ASLayout。
* 如果ASDisplayNode选择实现layoutSpecThatFits，由更为抽象的ASLayoutSpec来负责指定布局规则，由于ASLayoutSpec也实现了ASLayoutable接口，同样也可以通过调用measureWithSizeRange来获得ASLayout。

由于**ASLayoutSpec只负责指定布局规则，而不关心其布局的ASLayoutable具体是Node还是其他ASLayoutSpec**，我们可以轻松地将ASLayoutSpec生成逻辑独立出来，达到复用的目的。

#### 挑大梁的ASStackLayoutSpec

ASLayoutSpec主要有以下几种：

![](https://cdn-images-1.medium.com/max/1600/1*w0bazU9U5hj12AHi1glUgw.png)

通过它们的命名就可以了解其大致作用。其中用途最广泛的无疑是ASStackLayoutSpec，与UIStackView有异曲同工之妙。

ASStackLayoutSpec大量借鉴了CSS的FlexBox概念，写过Web代码的同学应该能一眼认出许多熟悉的属性：justifyContent/flexDirection/alignSelf/alignItems/flexBasis等等。不熟悉的朋友也可以通过一个有趣的游戏来学习flexbox：[http://flexboxfroggy.com/](http://flexboxfroggy.com/)

###示例
(图中的Huy Nguyen也是ASDK布局的主要贡献者之一)

![](https://cdn-images-1.medium.com/max/1600/1*YO94nNw73fuxuEvw5xeYbg.png)

图中左边是一个imageNode，右边是由两个textNode组成的一个vertical
stack，再和imageNode组合成一个横向的stack，最后加上边缘inset完成整个布局。

事实上，与css类似，绝大多数的布局都可以通过stack和inset的组合来完成；而和UIStackView不同的是，layout spec只是一个存在于内存之中的数据结构，并不需要额外创建view容器来承载子元素，大大节约了复杂布局带来的开销；同时因为它的轻量级和独立性，因此可以异步地计算，也可以缓存在node之中，对于大量tableViewCellNode快速布局有相当大的帮助。

采用Automatic Layout的优势是明显的：

* 借用css成熟的flexbox，有大量现成资料和案例来学习，也继承了它大量的优点。
* 由于ASDK是开源的，可以非常方便的调试。
* 层次清晰，即使是复杂布局也相对易于维护。
* 相似的布局规则容易复用。

Huy Nguyen也为ASLayoutSpec写了一个寓教于乐的[小游戏](http://nguyenhuy.github.io/froggy-asdk-layout/)，只是将上面提到的的针对css的小游戏版本改成了Objc，方便大家快速学习。
