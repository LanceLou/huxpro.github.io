---
layout:     post
title:      "最大最小算法之tic-tac-toe"
subtitle:   "Lance2016总结-算法篇(1)"
date:       2017-02-11 00:00
author:     "LanceLou"
header-img: "img/post-tictactoe-ai-banner.png"
catalog: true
tags:
    - 算法
    - 2016前端总结集
---

>博弈论考虑游戏中的个体的预测行为和实际行为，并研究它们的优化策略。表面上不同的相互作用可能表现出相似的激励结构（incentive structure），所以它们是同一个游戏的特例。 --wikipedia

<br>
<br>

2016年一直想做的一个小游戏，freecodecamp中的一个任务，碍于各种原因和借口，一直没有下手，这几天正好在总结2016，故也算是把这个任务给终结掉！

## 序
tic-tac-toe，又名井字棋，相关知识我就不做多介绍了。

在此我们主要解决计算机的下棋问题！

在此我希望大家明白一个道理，在棋类博弈中，任何一方的最终目的都是"保证自己不输的情况下来赢得棋局"。那么在这之中，就有主攻的，有主守的。何为主攻，围你城墙，困你士兵，使对方处于不利的地位，往往是最先下手的一方。何为主防，在敌人的围追堵截下求得一丝生机，留得青山在不怕没柴烧！具体到本文的tic-tac-toe，你会发现，其实攻中带防，防中带攻！

还有一点，计算机是如何思考的，具体到博弈中，计算机是如何决策的，计算机最擅长的就是做"重复"的事情，对于诸如AlphaGo及其神经网络是如何决策的我不知道，但有一种，也是本文所用到的决策的方法，即在敌我双方进行交替出棋的情况下，穷举所有的局(在内部穷举出所有可能的情况)，对各种情况进行现场评估，得出最优解。当然，此穷举非真正的穷举，一是要依据具体的计算能力进行适当终止，二是可以使用一些剪枝算法进行无意义的"穷举路径"的剔除！那么对于穷举的深度也就决定了计算机的智能程度(棋力)！

## 主体
#### 最大最小算法

这里插播一个小知识点，零和博弈 [Wikipedia](https://zh.wikipedia.org/wiki/%E9%9B%B6%E5%92%8C%E5%8D%9A%E5%BC%88)。 简介: 零和博弈表示所有博弈方的利益之和为零或一个常数，即一方有所得，其他方必有所失。其实我们这个三连棋就是一个零和博弈局。很突出的一点，一开始就是平局，这个平局说的是各自获胜的几率相等，在各个点下棋获胜的长期回报是一样的，不会因为双方不同的策略(下棋组合)而导致结果的变化。一方的获胜只有在其他方所失(犯错)的情况下发生！

棋盘(现场)评估: 针对当前棋盘上的X，O方棋子分布进行打分。这里作者给出六个状态:

* 0: 和局
* 1: 有待进一步对决
* 50: 两组成二的布局(对方)
* -50: 两组成二的布局(我方PC)
* 100: 一组成三布局(对方赢了)
* -100: 一组成三布局(我方PC赢了)

这里我们着重解释-50与50这种状态，单方面两组成二，什么叫单方面两组成二，我称之为无懈可击，就是如果我方处于这种状态，对方就无法翻盘了(成二: 两枚单方棋子成一线，还剩下一个空位)。在这里我们将单方面一组成二归为1，因为这种状态我们视为无法利于某一方，因为另一方可以马上将其堵住。拿到实际中，而不是电脑的思维，你下三连棋的时候，是不是也是这么想的？

其实下棋的时候，是我们的一个思考的过程，回顾一下，你决定落子的时候是如何想的，如果我下在这儿，我是否就能赢(马上，一步)，如果我不能赢，那我会去想，我的对手在我落子之后会落在哪儿? 我要保证它不先于我赢，然后就是保证我要赢！其实这两个保证是一样的。

* minimax: 从上方的评估函数，我们不难看出，值越小，对我方(PC)越有利。
* max: 在所有的落子选择中，选择对我方(PC)最不利，值最大的选择。
* min: 在所有落子选择中，选择对我方(PC)最有利，最最小的选择。

计算机每次决定落子的时候，就会建立一颗博弈树，这棵博弈树由两方交叉落子产生(穷举所有落子情况)；计算机如何抉择，其实和我们人的思维是一样的，求出各方的各自对应步骤的最优解，然后模拟各方的落子情况(都是最小化对方的最优解)。我们也只需要考虑对方的最优解，因为如果对方不按照我们的最优解下，对我们只有好处没有坏处！由于我们的两方共用一个评估函数，所以就出现了最大最小之说。

```js
	minSearch(oneDMap) {
		let positionValue = this[_gameState](oneDMap),
			i,
			bestValue = INFINITY;
		if (positionValue === DRAW) return 0;
		if (positionValue !== INPROGRESS) return positionValue;
		for (i = 0; i < 9; i++){
			if (oneDMap[i] === empty) {
				oneDMap[i] = this.curPcChees;
				let value = this.maxSearch(oneDMap);
				if (value < bestValue) 
					bestValue = value;
				oneDMap[i] = empty;
			}
		}
		return bestValue;
	}

	maxSearch(oneDMap) {
		let positionValue = this[_gameState](oneDMap),
			i,
			bestValue = -INFINITY;
		if (positionValue === DRAW) return 0;
		if (positionValue !== INPROGRESS) return positionValue;
		for (i = 0; i < 9; i++){
			if (oneDMap[i] === empty) {
				oneDMap[i] = Math.abs(1 - this.curPcChees);
				let value = this.minSearch(oneDMap);
				if (value > bestValue) 
					bestValue = value;
				oneDMap[i] = empty;
			}
		}
		return bestValue;
	}

	minimax(oneDMap) {
		let i,
			bestValue = INFINITY,
			index = 0,
			bestMove = [];
		for (i = 0; i < 9; i++){
			if (oneDMap[i] === empty) {
				oneDMap[i] = this.curPcChees;
				let value = this.maxSearch(oneDMap);
				console.log(value);
				if (value < bestValue) {
					bestValue = value;
					index = 0;
					bestMove[index] = i;
				}else if (value === bestValue) { //best position more than one
					bestMove[index++] = i;
				}
				oneDMap[i] = empty;
			}
		}
		if (index > 0) 
			index = Math.floor(Math.random() * index);
		return bestMove[index];
	}
	
```


#### Alpha-beta剪枝
	
一种剪枝技术，被用来精简博弈树。

思想: 边生成博弈树边计算评估各节点的倒推值，并且根据评估出的倒推值范围，及时停止扩展那些已无必要再扩展的子节点，即相当于剪去了博弈树上的一些分枝，从而节约了机器开销，提高了搜索效率

依据: 棋手不会做对自己不利的事情。

其实Alpha-beta剪枝也有各种不同的剪枝方式，这里只列举其中一种。

对于一个分支节点MIN，如果当前下确界未超过其父节点MAX的上确界，就没有不要在分支下去了。同理对于一个MAX的分支节点，如果当前上确界未小于其父节点MIN的上确界，也没有在不要在分支下去了；这像一种组合拳，去掉没有必要走的绕道。

```js
	minSearch(oneDMap, alpha, beta) {
		if (beta<=alpha){  
             return   evalValue;  
      	}  
		let positionValue = this[_gameState](oneDMap),
			i,
			bestValue = INFINITY;
		if (positionValue === DRAW) return 0;
		if (positionValue !== INPROGRESS) return positionValue;
		for (i = 0; i < 9; i++){
			if (oneDMap[i] === empty) {
				oneDMap[i] = this.curPcChees;
				let value = this.maxSearch(oneDMap, alpha, Math.min(bestValue, beta)));
				if (value < bestValue) 
					bestValue = value;
				oneDMap[i] = empty;
			}
		}
		return bestValue;
	}

	maxSearch(oneDMap, alpha, beta) {
		if (beta<=alpha){  
             return   evalValue;  
      	}  
		let positionValue = this[_gameState](oneDMap),
			i,
			bestValue = -INFINITY;
		if (positionValue === DRAW) return 0;
		if (positionValue !== INPROGRESS) return positionValue;
		for (i = 0; i < 9; i++){
			if (oneDMap[i] === empty) {
				oneDMap[i] = Math.abs(1 - this.curPcChees);
				let value = this.minSearch(oneDMap, Math.max(bestValue, alpha), beta);
				if (value > bestValue) 
					bestValue = value;
				oneDMap[i] = empty;
			}
		}
		return bestValue;
	}

	minimax(oneDMap) {
		let i,
			bestValue = INFINITY,
			index = 0,
			bestMove = [];
		for (i = 0; i < 9; i++){
			if (oneDMap[i] === empty) {
				oneDMap[i] = this.curPcChees;
				let value = this.maxSearch(oneDMap, -INFINITY, +INFINITY);
				console.log(value);
				if (value < bestValue) {
					bestValue = value;
					index = 0;
					bestMove[index] = i;
				}else if (value === bestValue) { //best position more than one
					bestMove[index++] = i;
				}
				oneDMap[i] = empty;
			}
		}
		if (index > 0) 
			index = Math.floor(Math.random() * index);
		return bestMove[index];
	}
```
	

## 结语
	
miniMax,也算是一种最大最小算法，算是作者在AI方面一点点皮毛到不要太皮毛的入门，虽然不是干AI这行，但感受一下不会太差。其实我个人还是觉得，计算机是死的，如何让他像人那么思考，我们如何想的，具体的思考步骤，一步步输入计算机，通过特定的数据结构，来存储，计算和思考。告诉计算机如何思考，如何想人这么思考。

当然，这只是作者的一点个人见解，可能太多不足之处，如有不足，欢迎各位指导。

## 引用

* [α-β剪枝技术](http://ist.csu.edu.cn/ai/Ai/chapter3/333.htm)
* [Minimax算法研究（TicTacToe](http://univasity.iteye.com/blog/1170216)
