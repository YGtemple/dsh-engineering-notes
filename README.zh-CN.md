<div align="right">

[English](README.md) · **[简体中文](README.zh-CN.md)**

</div>

# dsh-engineering-notes

在 DeepSeek Harness 里接入第三方服务的现场笔记——传输层调试、插件架构、Windows 部署修复。

<sub>Markdown · 工程笔记</sub>

---

## 里面有什么

让外部服务在 DeepSeek Harness 里跑通时记下的笔记，分三个领域：

| 领域 | 覆盖什么 |
|---|---|
| 传输层调试 | 请求在客户端与服务方之间哪个环节失败，以及怎么判断是哪一层的问题 |
| 插件架构 | 插件接口是怎么组织的，一个服务接入需要符合什么 |
| Windows 部署修复 | 修那些在 Windows 上因为环境而非代码而失败的安装 |

## 为什么要写下来

接入一个服务看起来像配置工作。其实不是——大多数失败发生在传输层或环境层，而报错信息会指向错误的地方。按症状把真正的原因记下来，下一次接入就不用重新推一遍。

## 格式

每篇笔记都是同一个形状：症状、看起来像什么、真正原因、怎么确认。最后一项重要——验证不了的修复只是猜测。

## 相关

- [agent-debug-notes](https://github.com/YGtemple/agent-debug-notes) —— agent 会话里的调试笔记
- [dsh-multi-router-intl](https://github.com/YGtemple/dsh-multi-router-intl) —— 这项工作最终服务的路由器
- [个人主页](https://github.com/YGtemple) —— 全部项目

## 许可证

MIT
