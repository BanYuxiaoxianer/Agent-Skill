# dsh-ctx-squeeze · 上下文挤压

> 31's context squeeze for the DeepSeek Harness Web GUI — 把一段长对话挤成几句话，开个干净的新会话接着干。

面向 **harness 0.2.0-rc.2** 的 Web 插件。会话标题栏多一颗「压缩」按钮：点一下，宿主把当前对话压成 3~10 条中文要点，然后在**同一个工作区新建会话**，把摘要作为第一条消息注入；原会话原封不动留在侧边栏当完整历史。

适合小上下文窗口的模型，或者聊太长、token 账单开始发烫的时候。

## 和内置压缩的区别

| | 内置 compaction-basic | dsh-ctx-squeeze |
|---|---|---|
| 位置 | 原地把历史压成 checkpoint | 新开一个会话，旧会话保留 |
| 触发 | 接近窗口上限时自动 | 手动点按钮，先确认 |
| 好处 | 上下文连续 | 旧会话可随时翻回去，新会话从头干净 |

两者都开没问题：内置压缩负责兜底，这个插件负责「我想重开一局」的时刻。

## 安装

```bash
dsh plugin --profile web add D:\GitHub\dsh-ctx-squeeze
```

装完重启 harness（`dsh web`），会话标题栏右侧出现按钮。

## 配置

profile 条目里可以覆盖（不写就用默认）：

```yaml
- id: ui-ctx-squeeze
  name: 'dsh-ctx-squeeze'
  config:
    enabled: true              # 总开关
    maxInputChars: 20000       # 重放给模型的对话字符上限，超了从最旧的开始丢
    maxTokens: 2048            # 摘要的 token 预算
    summarizationProvider: ''  # 留空 = 跟着源会话自己的 provider
    summarizationModel: ''     # 留空 = 跟着源会话自己的 model
```

> `summarizationProvider` 和 `summarizationModel` 必须**成对**填写，只填一个会被忽略。

## 工作流程

1. 前端 POST `/api/ctx-squeeze/run`，带上 `sessionId`。
2. 宿主读这个会话的事件日志，用会话原本的 system prompt、tools 和消息前缀重放出输入（这样能吃到 provider 的前缀缓存，不被当成新请求）。
3. 在重放内容后面追加一条压缩指令，走 `ctx.llm` 流式请求，只收文本块；被 token 上限截断直接判失败。
4. 新建会话（同 cwd、同 agentPreset，记 `parentSession`），把摘要包成 `<squeeze-checkpoint>` 消息写进去并 flush。
5. 前端刷新列表、跳到新会话，标题续成「原标题 · 续」。

源会话全程只读。密钥不出 harness，也不会进任何会话记录。

## 水印

水印只存在于源码，**界面与对话正文里都看不到**：

- `lib/core.js` 的 `BRAND_SIGN`，写在摘要首块的 `marker` 元数据上（不进可见正文）
- 每个源文件头部注释、`package.json` 的 `author`、`LICENSE` 的版权行
- 要换名字：改 `lib/core.js` 的 `BRAND_SIGN`、`package.json` 的 `author`、`LICENSE` 版权行即可

## 仓库结构

```
dsh-ctx-squeeze/
├── lib/core.js      # 纯函数：设置合并、守卫判断、重放、摘要包装（可单独测）
├── lib/index.js     # host：注册路由、调 ctx.llm、建新会话
├── lib/client.js    # 前端：手写 bundle，挂到会话标题栏 actions 座位
├── dsh-ctx-squeeze.yml  # profile 补丁（只 insert，不替换官方组合）
├── package.json
└── test/smoke.cjs   # node test/smoke.cjs
```

## 测试

```bash
cd D:\GitHub\dsh-ctx-squeeze
node test/smoke.cjs
```

不需要装 harness 依赖：host 的纯逻辑从 `lib/core.js` 直接测，前端用 vm + 假 React 跑一遍渲染和判断分支。

## 已知限制

- 只在模型空闲（没有进行中的 turn、没有未提交的输入、历史窗口已打开）时可用，忙时按钮禁用。
- 摘要质量取决于路由到的模型；用很便宜的模型压长对话可能丢细节。
- 压缩后原会话不会被删，历史多了要自己清。

## License

MIT © 31
