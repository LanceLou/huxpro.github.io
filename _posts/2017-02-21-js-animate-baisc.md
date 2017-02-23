---
layout:     post
title:      "JavaScript动画以及缓动公式"
subtitle:   "animation in JavaScript"
date:       2017-02-21 00:00
author:     "LanceLou"
header-img: "img/post-bg-js-version.jpg"
catalog: true
tags:
    - JavaScript
    - CodingAbilityPromote2017
---

> JavaScript动画，如何使用脚本语言写出动画？如何实现各种动画的缓动效果？今天，Lance就带各位来瞅瞅这JavaScript中的动画!


## 场景(S)
最近在敲一个轮播图组件，说实话，整体是没有太多坑的，但考虑到兼容IE8，在IE8中CSS animation无法支持，所以没有使用CSS3中的动画过度效果，自己敲了一个js动画过度，主要是用来进行图片的轮播展示！

关于这个组件的源码，大家可以移步我的GitHub:[https://github.com/LanceLou/CodingAbilityPromote2017/tree/master/lanceCarousel](https://github.com/LanceLou/CodingAbilityPromote2017/tree/master/lanceCarousel)

## 正题(T&A)

#### Carousel实现

好的，废话不多说，说说我这个组件，轮播图，大家用过，见过的都知道，有一个过度效果，图片从左边过度到右边，也即进入下一张。

那么如何实现这种图片的无缝过度，一张张的没有结束的感脚呢？实现起来其实很简单!

![Demo-1](/img/js-carousel-demo-1.png)

如图，相同色块为同一张图片，其实在这个轮播结构中，我们真正要展示的只有三张照片(item),也即图中红色框内的三张图片(img1, img2, img3)。其他两张为辅助用，如何让图片间的过度无缝呢，相邻之间的直接过度我们不多说，跳跃式的也类似，但像最后一张到第一张，第一张到最后一张这种边界性质的过度，就是我们这两张多出的图片的作用了: 很简单，比如上图，我们要从img3过度到img1，如何实现？我们先从img3过渡到最后一张img1(我称之为为预备的img1)，过渡完之后迅速调到第一张img1。

这里我需要说明下，过渡是指滑动的过程，你是可以看到过程的，有一个时间段；而像我上文提到的迅速跳到某处，是指中间无过渡，无滑动，表现到JavaScript中就是只直接给对应元素赋值目标值。过渡与迅速跳转的区别涉及一种视觉欺骗，视觉方面我是外行，这儿就不累赘了。

有同学可能会问，这个"动"是如何实现的，其实很简单，以上面Demo为例，这个5张照片外面我们包一个容器，让这个容器动起来，那么这个整体的Carousel的轮播就由这个容器承包了。

#### 滑动&缓动

比较关键的地方，如何实现平缓过度，又如何实现各种诸如ease-in, ease-out等的缓动效果？

首先我们来说平滑过度，无中断，无卡顿，自然。

JavaScript中处理位置的变化我们可以使用absolute定位的top，right，bottom，left的值得变化来实现，由于javascript处理位置变化太有效率，根本不可能让你有“移动”的感觉，感觉是直接从一点跳到某一点。那么如何让元素有滑动的感觉呢，这里引用***司徒正美***博客中的一段话

>我们必须让物体每移动一点点，就停一下，让眼睛有个残影。根据人眼睛的视觉停留效应，若前一幅画像留在大脑中的印象还没消失，后一幅画像就接踵而至，而且两副画面间的差别很小，就会有“动”的感觉。那么停留多么毫秒最合适呢？我们不但要照顾人的眼睛，还要顾及一下显示器的显示速度与浏览器的渲染速度。根据外国的统计，25毫秒为最佳数值。其实，这个数值我们应该当作常识来记住。联想一下，日本动画好像有个规定是1秒30张画，中国的，比较垃圾，是1秒24张。用1秒去除以张数，就得到每张停留的时间。日本的那个27.77毫秒已经很接近我们的25毫秒了，因为浏览器的渲染速度明显不如电视机的渲染速度，尤其是IE6这个拉后腿的。要实现加速度，就是让它每次移动快一点点，让上一次移动的距离乘以一个大于1的数便可。

总结来说: 就是25毫秒的间隔，而且移动的距离要尽量小。

举个例子(呆滞的平滑, gif图可能因pc即不同显示有卡顿)

```js
var container = document.querySelector(".lanceCarouse-imgContainer"),
	curLeft = -1320;
function animate(){
    container.style.left = curLeft+"px";
	curLeft = curLeft - 15;
    if(curLeft >= -3960)
        setTimeout(animate, 25);
}
setTimeout(animate, 25);
		
```

![demo-02](/img/js-carousel-demo-2.gif)

从上面的也是大家可以看到，如果采用匀速的过度，效果过分呆滞，没有生气。来点变速的？

```js
function animate(){
    container.style.left = curLeft+"px";
	speed = speed * 1.06;
	curLeft = curLeft - speed;
    if(curLeft >= -3960)
    	setTimeout(animate, 25);
    else
    	container.style.left = "-3960px";
}
```

设置加速度为1.06，其实不用我说，像这种简单的加速减速滑动，也不够优美，不过没关系，编程界大神们早就给我们造好轮子了，今天就列举一下Robert Penner大神的缓动公式，内置到tween类中。

```js
var Tween = {
    Linear: function(t,b,c,d){ return c*t/d + b; },
    Quad: {
      easeIn: function(t,b,c,d){
        return c*(t/=d)*t + b;
      },
      easeOut: function(t,b,c,d){
        return -c *(t/=d)*(t-2) + b;
      },
      easeInOut: function(t,b,c,d){
        if ((t/=d/2) < 1) return c/2*t*t + b;
        return -c/2 * ((--t)*(t-2) - 1) + b;
      }
    },
    //......还有很多，版面有限，此处不意义列举，感兴趣的同学可以自行搜索哦！
}
```
不要眼花，以上就是tween类，对于相关的参数，这里一一解释:

* t:timestamp，指缓动效果开始执行到当前帧开始执行时经过的时间段，单位ms
* b:beginning position，起始位置
* c:change，要移动的距离，就是终点位置减去起始位置。
* d: duration ，缓和效果持续的时间。

eg:

```js
function animateUpd(){
	var begin = 2640,
		end = 1320,
		change = -1320,
		duration = 1000,
		startTime = new Date().getTime();
	function animateFunc() {
		var newTime = new Date().getTime(),
			timestamp = newTime - startTime;
		container.style.left = "-"+ Tween.Quad.easeInOut(timestamp, begin, change, duration) + "px";
		if (duration <= timestamp) {
			container.style.left = "-" + end + "px";
		}else{
			setTimeout(arguments.callee, 25);
		}
	}
	setTimeout(animateFunc, 25);
}
```
看效果:

![Demo-3](/img/js-carousel-demo-3.gif)

## 总结(R)

使用JavaScript产出平滑过渡的动画效果，也算是get一个小技能，其实嘛，限于浏览器的兼容性，相比使用JavaScript来绘制动画，个人觉得使用CSS3的 ***animation*** 和 ***transition*** 可能会更胜一筹，毕竟算是浏览器的原生的，在性能和简洁方面要更好。



