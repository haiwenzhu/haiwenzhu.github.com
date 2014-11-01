---
layout: post
title: "将Blog改成了响应时"
categories:
  - HTML/CSS
---

其实用这个blog就写了两篇文章，可能是我道行尚浅，总觉得没有太多东西可写，有些东西，写出来肤浅，而有些东西写出来，又矫情，再加上懒惰，所以写的更少了。只能说三个字，要不能停。

前些时候，突然觉得应该把我的这个不怎么写的blog改成响应式的，于是上个周五的晚上，花了一晚上的时间改了一下，这个礼拜，花点时间记录一下。

所谓的响应式设计（[教程](http://www.smashingmagazine.com/2011/01/12/guidelines-for-responsive-web-design/ "参考")），其实就是根据访问站点的终端的尺寸，动态的调整页面的布局。而实现这个功能的关键就是media queries。media queries支持查询当前设备的屏幕尺寸（device-width），屏幕方向（orientation）等等。然后针对不同的尺寸设置不同的样式。举个例子：

	@media (max-width: 480px) {
		.small_screen {
			max-width: 100%;
		}
	}

上面这段css，.small_screen样式只会在宽度小于480px的时候有效。

注意这里有一个logical pixel和physical pixel的区别（[参考](http://ariya.ofilabs.com/2011/08/mobile-web-logical-pixel-vs-physical-pixel.html)），所谓的logical pixel就是css获取到的像素值，而physical pixel是屏幕的物理尺寸。两者并不一定相等，取决于你屏幕的分辨率。所以需要设置viewport这个meta flag让logical pixel适配physical pixel。通常的设置为

	<meta name=”viewport” content=”width=device-width initial-scale=1”>

而我在做的过程过发现即使设置了viewport，不同的浏览器表现也是不一样的。当我设置body的属性为`width:100%`时，在chrome下表现是正常的，但是在Android的原生浏览器下，body的宽度变成了屏幕宽度的两倍（我用得是note2，屏幕宽度是360，在Android原生浏览器下body宽度变成了720）。window.screen.width和window.innerWidth两个属性的值，前者是720，后者是360。google出来的结果大部分都是建议设置viewport，怎么都搞不定。

不得已只能用比较山寨的方法，用js判断scree.width和innerWidth两个值是否相等，不等的话就设置body的宽度为较小的值。设置了style之后发现在Android原生浏览器下还是没有效果。设置的宽度没有生效，理论上内联样式的优先级是最高的，不知道怎么在Android原生浏览器下优先级反而没有class的优先级高，设置了`!important`才达到预期的效果。