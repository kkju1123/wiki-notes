# Medium 的“Just a moment...”安全验证页：Cloudflare 反爬虫机制解析

*原文: [https://medium.com/@tauhidnoor/the-differences-between-an-encoder-decoder-model-and-decoder-only-model-76f56e336378](https://medium.com/@tauhidnoor/the-differences-between-an-encoder-decoder-model-and-decoder-only-model-76f56e336378) · 来源: medium · 生成时间: 2026-09-22T07:36:32.843068+00:00*

## 背景

很多内容网站（如 Medium、GitHub、Pixiv）会在边缘部署安全服务，以防止恶意爬虫、DDoS 攻击、撞库和数据剽窃。Cloudflare 是其中最常见的服务商，其“Just a moment...”页面就是在浏览器挑战通过前展示的中间页。它出现说明请求被边缘节点接管，正在验证客户端是否为真实浏览器。

## 痛点

对普通用户它通常一闪而过，但对爬虫和自动化工具来说是常见卡点：requests/curl 不执行 JavaScript，会永远停留在验证页或收到 403。如果开发者不理解这一层，容易误判为目标站点宕机或 IP 被封，浪费大量调试时间。

## 解决办法

Cloudflare 通过 JavaScript Challenge 验证客户端。服务器返回一个包含 JS 代码的 503/403 页面，浏览器执行该 JS 后生成一个一次性 token（如 __cf_bm 或 cf_clearance cookie），再携带该 cookie 重新请求。同时 Cloudflare 会检查 TLS 指纹、HTTP/2 指纹、User-Agent、屏幕分辨率、navigator.webdriver 等特征做综合 Bot 评分。类比机场安检：先用自助闸机刷证件（JS 计算），再通过安检门（行为特征），通过后才允许进入登机口。

## 关键代码示例

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)  # 避免被检测为无头浏览器
    page = browser.new_page()
    page.goto("https://medium.com/example-article")
    page.wait_for_load_state("domcontentloaded")
    print(page.title())  # 通过后不再是 "Just a moment..."

```

普通 requests 等 HTTP 客户端不执行 JavaScript，无法完成 Cloudflare 的挑战；Playwright 驱动真实 Chromium 执行 JS、保存 Cookie，并带上浏览器指纹，所以能通过大多数基础验证。headless=False 是为了降低被 Cloudflare 识别为自动化工具的概率；实际生产中应优先使用官方 API 或 RSS，并控制抓取频率。

## 关键流程

1. 用户请求到达 Cloudflare 边缘节点，节点根据 IP、请求头、TLS 指纹等做初步评分
2. 若评分可疑，返回 503/403 状态码和一段 JavaScript 挑战页面，页面上显示“Just a moment...”
3. 真实浏览器执行 JS 计算并生成一次性 token/cookie（如 __cf_bm、cf_clearance），自动重试请求
4. Cloudflare 校验 token、cookie 与浏览器特征，通过后将请求转发给源站 Medium
5. 源站返回真实内容，浏览器正常渲染；未通过则继续拦截或要求验证码

## 关键点

- 出现“Just a moment...”页面说明请求被 Cloudflare 边缘节点拦截，而不是 Medium 宕机；这是排障的第一判断点。
- 该验证的核心是 JavaScript Challenge：服务器返回 JS，要求客户端执行并回写 Cookie，所以 requests/curl 这类不执行 JS 的客户端无法通过。
- Cloudflare 不仅看 Cookie，还会综合 TLS 指纹、HTTP/2 指纹、UA、屏幕分辨率、navigator.webdriver 等特征做 Bot 评分。
- 抓取 Medium 等内容平台时优先使用官方 API/RSS；如果必须抓 HTML，需要使用能执行 JS 的浏览器自动化工具，并控制频率。
- 可以通过响应头中的 server: cloudflare 和页面文本 Just a moment 来快速识别 Cloudflare 挑战，避免误判。
- 验证 Cookie 具有时效性且与 IP/User-Agent 绑定，手动复制 Cookie 只能短期使用，不适合稳定采集。

## 对比与权衡

- 相比传统交互式验证码（如 reCAPTCHA），Cloudflare 的 JS Challenge 对正常用户无感、流失率低，但对高级爬虫的拦截强度较弱。
- 相比单纯 IP 黑名单/WAF 规则，Cloudflare Bot Management 通过边缘节点和指纹评分能识别分布式爬虫，但配置灵活性和可解释性不如自建 WAF 规则。
- 相比 Akamai 的 Bot Manager，Cloudflare 生态更易用、文档更多，但在大型金融企业级场景的成熟度和定制能力上不如 Akamai。

## 自测问题

**问: 为什么访问 Medium 会看到“Just a moment...”？**

说明请求被 Cloudflare 边缘节点拦截，进入浏览器挑战。Medium 启用了 Cloudflare 安全服务来防爬虫和攻击，正常浏览器会自动通过，通常无感。

**问: 用 Python requests 爬 Medium 时遇到 403 或验证页怎么办？**

先判断是否被 challenge；优先看 Medium 官方 API/RSS；若必须解析 HTML，换用 Playwright/有头浏览器，并处理 cookie、频率和 robots.txt，不要硬碰。

**问: Cloudflare 的 JS Challenge 具体怎么工作？**

服务器返回 JS 代码，浏览器执行后计算 token，写到 Cookie 中（如 __cf_bm），然后自动重试；服务端校验 token 及浏览器环境，通过后放行。

**问: 为什么有的无头浏览器也会被拦？**

Cloudflare 检测 navigator.webdriver、无头 Chrome 特征、自动化框架注入的变量等，即使执行了 JS 也可能被判定为机器人；需要隐藏这些特征或使用真实浏览器。

**问: 能否直接把浏览器里的 cf_clearance Cookie 复制给 requests 用？**

短期可行，但该 Cookie 与 IP、User-Agent、TLS 指纹绑定，有有效期，更换网络或 UA 后容易失效，且人工复制不适合规模化采集。

## 适用场景

- 调试爬虫/自动化工具时，遇到 Medium 等站点返回“Just a moment...”，需判断是否被反爬拦截。
- 面试中解释现代网站反爬虫原理，尤其是 Cloudflare 的 JS Challenge 和 Bot 评分机制。
- 评估第三方站点是否使用了 Cloudflare 防护，以决定采集方案和合规风险。
- 网站安全运维选型：了解 Cloudflare 防护如何在用户体验和反爬能力之间平衡。

## 标签

`Cloudflare` `反爬虫` `安全验证` `Medium` `爬虫`
