# Agent Note: SDK 会话控制

Status: implemented

[English](2026-08-17-sdk-session-control.md) | 中文

## 问题

agent 接口支持同轮次 steering（中途引导）和协作式取消，但子进程 SDK 客户端只能将提示词排入下一轮次或终止整个运行时。自动化所有者无法在不等待下一轮次的情况下改变运行中轮次的方向，而停止一个轮次会销毁该运行时托管的所有会话。

## 决策

SDK 协议在不改变 [agent 控制操作](../../../../packages/core/agent/README.md#agent-interface-typests)的情况下暴露 `session/steer` 与 `session/cancel`。JSON-RPC 服务器将获接受的 steering 路由到 `agent.steer()`，将获接受的取消路由到 `agent.cancel({ kind: 'user' })`；TypeScript 与 Python SDK 暴露对应的低层和高层方法。

## 协议语义

`session/steer` 接受与 `session/prompt` 相同的内容块，在需要时惰性创建会话，并返回持久 `MessageId`。该消息进入会唤醒 agent 的下一步骤 inbox，因此运行中的 agent 会在当前轮次最近的后续步骤边界消费该消息，空闲 agent 则为其开启一个轮次。

`session/cancel` 绝不创建会话。只有服务器观察到实时运行中的 agent 并请求协作式取消时，它才返回 `{ accepted: true }`；未知和空闲会话返回 `{ accepted: false }`。接受并不是完成结果，因此调用方通过 `session.event` 与 `session.status` 观察完全停稳。

提示词、steering 与取消仍是相互独立的请求。协议不把后续助手消息或 `turn/end` 事件归属于单个输入，也仍不提供逐会话关闭操作。

## 已考虑的替代方案

**把 steering 编码为另一个 `session/prompt`。** 这种做法会丢失下一步骤与下一轮次的区别，并使同轮次方向改变依赖队列时机，因此无法保留 agent 接口语义。

**通过关闭运行时子进程取消。** 这种做法会停止无关会话、放弃进程级复用，并把协作式轮次控制变成基础设施故障。

**在 agent loop（智能体循环）中增加控制行为。** 该循环已经拥有所需操作；重复实现会产生第二套生命周期实现，而不是在 SDK 协议中暴露现有操作。

## 后果

SDK 客户端可以改变活动工作的方向并将其停止，同时保留运行时及其其他会话。协议增加两个预发布请求/结果对，两种 SDK 实现及其文档同步演进；由于两项操作都不提供提示词级完成结果，客户端仍须拥有活动区间。

## 测试

服务器测试固定下一步骤路由、实时 agent 校验、仅运行中取消、未知会话行为和关闭期间拒绝。TypeScript 与 Python 子进程测试固定两层客户端，以及 TypeScript 对非法回执的处理。
