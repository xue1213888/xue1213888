---
title: "go-grpc文档使用笔记"
date: 2021-08-10T14:52:05+08:00
tags: ["rpc","微服务","grpc"]
categories: ["golang","微服务"]
draft: false
---

## grpc.Code
判断err是否是rpc系统生成的err

## grpc.ErrorDesc
获取err的string描述

## grpc.Errorf
生成一个rpc系统的error，带有rpc的code

## grpc.Invoke
发送一个rpc请求，一般都是生成的代码用来发送请求的。
arg1 .context 上下文
arg2 .method 就是方法在rpc方法，比如server.hello
arg3 req 请求的数据结构，作为一个interface是为了不受类型约束
arg4 reply 返回数据的结构
arg5 clientconn，一个客户端链接
arg6 opts calloption

返回错误

## grpc.Method
返回方法名，格式为 /服务器/方法名  /hello.HelloServer/SayHello

## grpc.

