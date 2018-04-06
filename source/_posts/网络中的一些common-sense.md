---
title: 网络中的一些common sense
date: 2017-03-06 17:29:57
tags: 5-tuple
categories: Networking
---

# 网络包需要另外20Ｂ
以太网帧需要１.前导码和帧开始符preamble(8B)，2.gap（12B）两帧之前的间隔。总共20B。
所以实际传输的最小帧为64B+20B=84B,
10Gbps的网，传输一个64B的数据包，需要84B/10Gbps=84*8bit/10*10^9bps=67.2ns

<!-- more -->

# 网络5-tuple
1. source IP address
2. destination IP address
3. network or transport protocol id
4. source port
5. destination port
