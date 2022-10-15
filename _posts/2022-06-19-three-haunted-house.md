---
layout: post
title:  "Three.js 之 11 Haunted House 恐怖鬼屋"
categories: Three.js
tags:  Three.js WebGL
author: Shen-Xmas
---

* content
{:toc}

本系列为 [Three.js journey](https://threejs-journey.com/) 教程学习笔记。

本节将使用我们之前学习的内容来创建一个鬼屋。我们会创建一个房子，有门、屋顶、和一些灌木，我们也会创建一些墓碑，还有幽灵的光飘过并产生投影。


本节完成效果，在线 [demo 链接](https://gaohaoyang.github.io/threeJourney/17-haunted-house/)

可扫码访问

二维码 | 手机截图
--- | ---
![](https://gw.alicdn.com/imgextra/i1/O1CN01LLkkbc1dluhbtAtt4_!!6000000003777-2-tps-200-200.png) | ![](https://gw.alicdn.com/imgextra/i2/O1CN01p7NrBp1gsjRfaPGeE_!!6000000004198-2-tps-750-1334.png)

[demo 源码](https://github.com/Gaohaoyang/threeJourney/tree/main/src/17-haunted-house)

开始之前先约定一下关于长度单位的问题。

根据不同场景，我们可以认为1代表的长度不同，例如创建比较宏大的场景如陆地地图可以认为1代表1km，创建房屋可以认为1代表1m，创建小场景可以认为1代表1cm。接下来就开始吧

# 创建房屋

## 地面和墙壁

使用群组的方式来添加房屋，为了后续方便整体调整房屋大小

```js
// house
const house = new THREE.Group()
scene.add(house)

// walls
const walls = new THREE.Mesh(
  new THREE.BoxGeometry(4, 2.5, 4),
  new THREE.MeshStandardMaterial({ color: '#ac8e82' })
)
walls.position.y = 1.25
