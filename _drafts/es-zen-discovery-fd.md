---
layout: post
title: Elasticsearch Zen Discovery 节点探活
categories: [Middleware]
description: 介绍 Elasticsearch Zen Discovery 的实现机制
keywords: Elasticsearch， Zen Discovery，Fault Detection
---

# Overview

Es在选主结束后，开始节点探活。节点探活分为两部分：非 master 节点 ping master节点，探测 master 节点的活性。master 节点 ping 其他节点，探活其他各个节点的活性。

# Master Fault Detection


# Node Fault Detection
