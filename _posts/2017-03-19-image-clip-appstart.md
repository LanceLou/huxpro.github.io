---
layout:     post
title:      "图片剪辑应用-技术实践"
subtitle:   "canvas, NodeJS, mogonodb etc"
date:       2017-03-19 00:00
author:     "LanceLou"
header-img: "img/post-image-clip-appstart.png"
catalog: true
tags:
    - NodeJS初体验
    - JavaScript
---


> 总想写点什么，遇到些许麻烦，但总归，路在脚下，梦在远方。
> 
> 不是~，还有诗和远方嘛！

OK, 入正题入正题，图片剪切应用。前几天，不对,前一两周做的小东西，但总归，也是做出来了，bug 些许，瑕疵些许，这些都是作者有待提升的，还望指导。

就直接上图了:

![Demo](https://www.tuchuang001.com/images/2017/03/20/imgClipCapture.gif)

## 知识点

在做这个应用时，有用到一些图形图像处理，ajax文件上传，Node数据存储等技术，特在此分享。

#### Blob对象

> JavaScript中一种表示二进制数据的对象，表示一个不可变的, 原始数据的类似文件对象。Blob表示的数据不一定是一个JavaScript原生格式。 File 接口基于Blob，继承 blob功能并将其扩展为支持用户系统上的文件。

几乎是一个与文件操作脱不了干系的，其本身没有多大意思，但却是一个JavaScript中二进制数据处理的桥梁，为用于二进制数据的大量JavaScript API提供重要的数据交换机制。在ImageClip应用中，就有应用场景。

1. 本地文件系统 -> Canvas (File -> Blob)
2. Canvas -> Web(Blob -> Ajax文件上传)

#### FormData

好东西，ajax文件上传必备，使用键值来模拟表单控件，其中就包括了File。

```JavaScript
var formdata = new FormData();
for(var name in data){
	if (!data.hasOwnProperty(name)) continue;
	var value = data[name];

	if (typeof value === "function") continue;
	formdata.append(name, value);
}
```
使用append方法将相应的键值(表单控件)添加进FormData。

具体读者可见: [https://developer.mozilla.org/zh-CN/docs/Web/API/FormData](https://developer.mozilla.org/zh-CN/docs/Web/API/FormData)

#### CanvasRenderingContext2D.drawImage()

咱们Canvas图形图像处理的关键，反正我觉得是这个图片剪切的核心，其实单看方法名还体现不出来，它有三种传参方式:

* void ctx.drawImage(image, dx, dy);
* void ctx.drawImage(image, dx, dy, dWidth, dHeight);
* void ctx.drawImage(image, sx, sy, sWidth, sHeight, dx, dy, dWidth, dHeight);

注意此处不要被image懵逼: 

> image: 绘制到上下文的元素。允许任何的 canvas 图像源(CanvasImageSource)，例如：HTMLImageElement，HTMLVideoElement，或者 HTMLCanvasElement。(可以是图像，视频，甚至是Canvas画布也可以)

关于图像剪切，我想读者应该没有多大障碍了吧。 好嘞，继续BB。

#### HTMLCanvasElement.toBlob()

这个方法可以根据canvas的当前内容生成一个Blob对象

#### URL.createObjectURL()

URL.createObjectURL() 静态方法会创建一个 DOMString，它的 URL 表示参数中的对象。这个 URL 的生命周期和创建它的窗口中的 document 绑定。这个新的URL 对象表示着指定的 File 对象或者 Blob 对象。

在使用图片显示Canvas输出的Blob时用到。

## 技术点

说了知识储备，说说技术点，

#### 拖动控件

![demo-1](https://www.tuchuang001.com/images/2017/03/20/drag-ctl.png)

布局应该没问题！

```JavaScript
<div class="clipUtil">
	<span class="dot lt"></span>
	<span class="dot mt"></span>
	<span class="dot rt"></span>
	<span class="dot lm"></span>
	<span class="dot rm"></span>
	<span class="dot lb"></span>
	<span class="dot mb"></span>
	<span class="dot rb"></span>
	<i class="horizonRect"></i>
	<i class="verticleRect"></i>
</div>
```

其实主要是这个大小调节问题，作者的思路是事件委托这个工具，然后监听这个上面的mousedown与mouseup事件，判断mousedown触发时作用的点(左上，上，右上，右，右下，下，左下，左)，mouseup时计算起点与重点的x，y方向相对位移，通过mousedown时触发的点来判断x，y的有效性，以及剪切工具的left，top。

大概就是如下这个方法了。

```JavaScript
/**
 * 剪切工具拖动放大移动计算
 * @param {String} direction 拖动方向，即拖的哪个点
 * @param {Object}   startPos  鼠标起始拖动位置
 * @param {Object}   endPos  鼠标结束拖动位置
 */
[_mouseDragMoveHandler](direction, startPos, endPos) {
	var toWidth = this.canvas.width,
		toHeight = this.canvas.height,
		toLeft = this.clipUtil.style.left,
		toTop = this.clipUtil.style.top,
		deltaX = endPos.x - startPos.x,
		deltaY = endPos.y - startPos.y;
	toLeft = toLeft.slice(0, toLeft.lastIndexOf("px")) - 0;
	toTop = toTop.slice(0, toTop.lastIndexOf("px")) - 0;
	switch(direction){
	 	case "lt":
	 		toWidth = toWidth - deltaX;
	 		toHeight = toHeight - deltaY;
	 		toLeft = toLeft + deltaX;
	 		toTop = toTop + deltaY;
	 		break;
	 	case "mt":
	 		toHeight = toHeight - deltaY;
	 		toTop = toTop + deltaY;
	 		break;
	 	case "rt":
	 		toWidth = toWidth + deltaX;
	 		toHeight = toHeight - deltaY;
	 		toTop = toTop + deltaY;
	 		break;
	 	case "lm":
	 		toWidth = toWidth - deltaX;
	 		toLeft = toLeft + deltaX;
	 		break;
	 	case "rm":
	 		toWidth = toWidth + deltaX;
	 		break;
	 	case "lb":
	 		toWidth = toWidth - deltaX;
	 		toHeight = toHeight + deltaY;
	 		toLeft = toLeft + deltaX;
	 		break;
	 	case "mb":
	 		toHeight = toHeight + deltaY;
	 		break;
	 	case "rb":
	 		toWidth = toWidth + deltaX;
	 		toHeight = toHeight + deltaY;
	 		break;
	}
	if (toWidth <= 60 || toHeight <= 60) return;
	this.updateClipParamAndDraw(toWidth, toHeight, toTop, toLeft);
	this[_triggerAllClipUtilAdjustListener]({top: toTop, left: toLeft, width: toWidth, height: toHeight});
}
```

#### 剪区提取

剪区提取是个比较麻烦的事儿，其场景是这样的:

我们的剪切工具(ClipUtil)与目标图像(Target)的状态的位置以及包含关系有许多，简单的如包含，无重叠，以及:::

![demo-3](https://www.tuchuang001.com/images/2017/03/20/ClipArae-demo.md.png)

作者暂时没有想到特别好的处理办法，SO， 分情况讨论:

```JavaScript
/**
 * 根据clipTarget与剪切工具的相对位置，长宽计算出剪切工具绘制其实坐标，长宽，剪切对象起始坐标
 * @return {Object} {cx，cy: 画布起始坐标 | width, hright: 绘制长宽，即两矩形重合区域 | ix, iy: 图片起始坐标，从图片的那个位置开始切}
 */
[_getClipUtilDramParams]() {
	var clipTaregtRect = this.clipTaregt.getBoundingClientRect(),
		utilRect = this[_clipUtilRect],
		deltaX = 0,
		deltaY = 0,
		pos = {}; /*pos: 剪切工具绘制参数: 画布其起始cx,cy 图片起始ix,iy 图片绘制宽度width, height*/

	// 图片不在剪接区域
	if (clipTaregtRect.left >= utilRect.right || clipTaregtRect.right <= utilRect.left) 
		return false;
	if (clipTaregtRect.top >= utilRect.bottom || clipTaregtRect.bottom <= utilRect.top) 
		return false;

	//-----------》》》》》》下面是图片与剪切区域有交集的情况
	
	//剪切区域包裹图片
	if (utilRect.top <= clipTaregtRect.top && utilRect.bottom >= clipTaregtRect.bottom && utilRect.left <= clipTaregtRect.left && utilRect.right >= clipTaregtRect.right) {
		return {cx: clipTaregtRect.left - utilRect.left, cy: clipTaregtRect.top - utilRect.top,
			ix: 0, iy: 0, width: clipTaregtRect.width, height: clipTaregtRect.height};
	}
	
	//图片包裹剪切区域
	if (clipTaregtRect.top <= utilRect.top && clipTaregtRect.bottom >= utilRect.bottom && clipTaregtRect.left <= utilRect.left && clipTaregtRect.right >= utilRect.right) {
		return {cx: 0, cy: 0, ix: utilRect.left - clipTaregtRect.left, iy: utilRect.top - clipTaregtRect.top, width: utilRect.width, height: utilRect.height};
	}

	//图片竖直穿过剪切区域
	if (clipTaregtRect.top <= utilRect.top && clipTaregtRect.bottom >= utilRect.bottom) {
		if (clipTaregtRect.left >= utilRect.left && clipTaregtRect.right <= utilRect.right) {
			//处在中间
			return {cx: clipTaregtRect.left - utilRect.left, cy: 0, ix: 0, iy: utilRect.top - clipTaregtRect.top, width: clipTaregtRect.width, height: utilRect.height};
		}else if (clipTaregtRect.left < utilRect.left) {
			//处在左边
			return {cx: 0, cy: 0, ix: utilRect.left - clipTaregtRect.left, iy: utilRect.top - clipTaregtRect.top, width: clipTaregtRect.right - utilRect.left, height: utilRect.height};
		}else{
			//处在右边
			return {cx: clipTaregtRect.left - utilRect.left, cy: 0, ix: 0, iy: utilRect.top - clipTaregtRect.top, width: utilRect.right - clipTaregtRect.left, height: utilRect.height};
		}
	}

	//图片水平穿过剪切区域
	if (clipTaregtRect.left <= utilRect.left && clipTaregtRect.right >= utilRect.right) {
		if (clipTaregtRect.top >= utilRect.top && clipTaregtRect.bottom <= utilRect.bottom) {
			//处在中间
			return {cx: 0, cy: clipTaregtRect.top - utilRect.top, ix: utilRect.left - clipTaregtRect.left, iy: 0, width: utilRect.width, height: clipTaregtRect.height};
		}else if (clipTaregtRect.top < utilRect.top) {
			//处在上边
			return {cx: 0, cy: 0, ix: utilRect.left - clipTaregtRect.left, iy: utilRect.top - clipTaregtRect.top, width: utilRect.width, height: clipTaregtRect.bottom - utilRect.top};
		}else{
			//处在下边
			return {cx: 0, cy: clipTaregtRect.top - utilRect.top, ix: utilRect.left - clipTaregtRect.left, iy: 0, width: utilRect.width, height: utilRect.bottom - clipTaregtRect.top};
		}
	}

	//左上
	if (clipTaregtRect.left < utilRect.left && clipTaregtRect.right > utilRect.left && clipTaregtRect.top < utilRect.top && clipTaregtRect.bottom > utilRect.top) {
		return {cx: 0, cy: 0, ix: utilRect.left - clipTaregtRect.left, iy: utilRect.top - clipTaregtRect.top, width: clipTaregtRect.right - utilRect.left, height: clipTaregtRect.bottom - utilRect.top};
	}

	//左下
	if (clipTaregtRect.left < utilRect.left && clipTaregtRect.right > utilRect.left && clipTaregtRect.top < utilRect.bottom && clipTaregtRect.bottom > utilRect.bottom) {
		return {cx: 0, cy: clipTaregtRect.top - utilRect.top, ix: utilRect.left - clipTaregtRect.left, iy: 0, width: clipTaregtRect.right - utilRect.left, height: utilRect.bottom - clipTaregtRect.top};
	}

	//右上
	if (clipTaregtRect.left < utilRect.right && clipTaregtRect.right > utilRect.right && clipTaregtRect.top < utilRect.top && clipTaregtRect.bottom > utilRect.top) {
		return {cx: clipTaregtRect.left - utilRect.left, cy: 0, ix: 0, iy: utilRect.top - clipTaregtRect.top, width: utilRect.right - clipTaregtRect.left, height: clipTaregtRect.bottom - utilRect.top};
	}

	//右下
	if (clipTaregtRect.left < utilRect.right && clipTaregtRect.right > utilRect.right && clipTaregtRect.top < utilRect.bottom && clipTaregtRect.bottom > utilRect.bottom) {
		return {cx: clipTaregtRect.left - utilRect.left, cy: clipTaregtRect.top - utilRect.top, ix: 0, iy: 0, width: utilRect.right - clipTaregtRect.left, height: utilRect.bottom - clipTaregtRect.top};
	}
}
```

如上，提取剪区参数，说白了，就是提取剪切对象的起始位置，剪区长宽，目标画布(ClipUtil)的起始位置，如有更好的处理办法，欢迎与作者讨论，还请不吝赐教。

#### 后台Node处理

上传的文件存文件系统，生成URL存mongodb, NodeJS刚起步，写的比较shit [ (￣.￣) ], 这就不班门弄斧了！

## 结语

至此，图片剪切一些技术问题也算梳理清了。如于你有用，my pleasure！ 如有不足之处，还请多多指教！

the end

———— 2017年03月 LanceLou