# 给 Wazuh 告警加个"中文翻译"：一个油猴脚本的诞生

> 安全运营每天面对海量英文告警，看一条都要反应半天。我写了个浏览器脚本，在 Wazuh dashboard 上自动把告警翻译成中文，顺便把 MITRE 技战术也标出来。

## 一、痛点

Wazuh 是开源的 SIEM/XDR，告警信息全英文，字段还特别多：

```
rule.description: Maximum authentication attempts exceeded.
rule.mitre.technique: Brute Force
rule.mitre.tactic: Credential Access
```

老手还好，新手（尤其非英语环境）每条告警都要反应几秒"这是啥攻击、多严重、怎么处置"，看多了容易漏。

## 二、方案：一个油猴脚本

我写了个 Tampermonkey 脚本，**只翻译"定性信息"，不碰纯标识字段**：

- `rule.description`（攻击描述）→ 中文
- `rule.mitre.technique` / `rule.mitre.tactic`（MITRE 技战术）→ 中文
- `agent.name`、IP、端口、id、groups 这类纯标识 → 不动（避免噪音）

效果：

```
rule.mitre.technique: Brute Force 🔍 暴力破解
rule.mitre.tactic: Credential Access 🔍 凭据访问
rule.description: Maximum authentication attempts exceeded. 🔍 SSH认证超限(暴力破解)
```

右下角还有个「📖 Wazuh 中文」速查面板，点开就是完整中英对照。

## 三、实现原理

核心就三点，代码不复杂：

1. **关键词 + MITRE 映射表**：内置攻击描述关键词（SSH 爆破/SQL注入/恶意软件…）和 MITRE 技战术中英对照表，命中就追加中文标签
2. **DOM 文本扫描**：用 `TreeWalker` 遍历页面文本节点，命中关键词就加标签，不破坏页面结构
3. **动态适配**：Wazuh 是 React 单页应用，内容动态渲染，用 `MutationObserver` + 防抖，页面变化后自动重扫；用 `dataset` 标记去重，避免标签叠加

## 四、安装使用

1. 浏览器装 Tampermonkey（Chrome/Edge 商店）
2. 打开 `edge://extensions` → Tampermonkey → 开启「允许用户脚本」
3. 新建脚本，粘贴代码，保存
4. 刷新 Wazuh dashboard，告警自动带中文

## 五、踩过的坑（比写代码本身更有意思）

1. **MutationObserver 无限循环**：脚本扫描时加标签，又触发自己的监听器，反反复复，直接把 Wazuh 页面卡在 Loading。解决：`dataset` 去重 + 800ms 防抖
2. **误标严重**：一开始把 `rule.mitre.tactic: Credential Access` 这种短字段值也当"告警"标了"未识别"，满屏噪音。解决：**只翻译"定性信息"（描述 + MITRE 值），纯标识字段不碰**
3. **翻译范围拿捏**：要翻什么、不翻什么，是从告警 JSON 结构里想清楚的——`rule.description` + `rule.mitre.*` 才是"定性"的，其他是"标识"的

## 六、局限

当前靠关键词 + MITRE 映射，覆盖常见攻击类告警。Wazuh 规则有几千条，全量覆盖得对接 Wazuh API 拉规则列表做完整对照表（这是下一步）。

## 七、结语

这个脚本本质是"把人工看英文告警的成本，用几十行 JS 自动化掉"——安全运营里这种"翻译/提效"的小工具很多，值得动手写。工具虽小，但**先明确"要翻译什么、不翻译什么"（定性 vs 标识），再谈实现**，这个思路比代码更重要。

代码开源在 GitHub：https://github.com/loadthing/wazuh-alert-zh-helper
