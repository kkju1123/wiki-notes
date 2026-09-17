---
title: Codex 最推荐的15个skill，每一个都值得收藏！
url: https://x.com/XiaohuiAI666/status/2095130302381494664
source_type: twitter
author: XiaohuiAI666
tags:
- Codex
- AI Agent
- Skill
- 开发效率
- 自动化
summary: 拆解 Codex 技能机制与 15 个高频 Skill，覆盖开发流程、调试、浏览器自动化与知识蒸馏，帮助读者按痛点选装并沉淀可迁移技能库。
fetched_at: '2026-09-17T01:42:47.584223+00:00'
---

大家好，我是程序员小灰。

如果说2023-2025年是AI大模型的三年，那2026年必然属于AI智能体。

如今国内外已经有各种各样优秀的AI智能体脱颖而出，比如国内的Workbuddy，国外的Codex，都为我们的工作和生活带来了数不尽的便利。

说起AI智能体，就离不开skill的讨论。

前一段时间，小灰刚刚分享了WorkBuddy 最值得推荐的 15 个技能，而今天，我们再来说一说Codex 最值得推荐的 15个 skill。

文章比较长，建议大家先收藏、不迷路。

## 一、首先搞懂 Codex 技能机制

到底什么是 skill（技能）呢？

所谓skill，可以理解成 给 AI 安装的插件，一份说明书就是一个技能，教它一项新本事：写代码前先出计划、报错了自动排查、替你操作浏览器。裸的 AI 像个聪明但两手空空的实习生，装上技能，就像给了他一柜子趁手的工具。

skill 的本质又是什么？

一个文件夹，里面一份 SKILL.md（名字、用途、执行流程），加可选的脚本和参考文件。Codex 启动时把每个技能的名字和描述预加载进系统提示词，干活时自己判断用哪个。

安装的位置分两种：~/.codex/skills/ 全局生效； .codex/skills/ 或 .agents/skills/ 只对该项目生效。不同版本对路径支持略有差异，装完问一句"你现在有哪些技能"，识别了才算装上。

官方技能目录在 openai/skills：系统内置区的五个技能随 Codex 自动装，精选区有 39 个技能用 $skill-installer 按名安装，覆盖开发框架、Figma 设计转代码、一键部署、Notion、安全审查、GitHub 工作流和 PDF/浏览器/截图/语音等工具类，是 Codex 的"开箱即用能力包"，遇到对应场景自动按官方最佳实践执行。

需要注意的是，每个技能都会消耗模型上下文窗口，安装数量过多，会导致模型选择技能时判断失准，该现象在大部分Agent工具中都存在。

从哪里寻找 skill 呢？

Codex 的技能市场有所不同，没有几十万个技能的大卖场，官方精选区（curated）里总共 39 个精选技能。对比 WorkBuddy 那边 7 万多个社区技能、3000 万次下载的货架，这点库存，顶多算个精品店柜台。

Codex 走 http://agentskills.io 开放标准，一个文件夹加一份 SKILL.md 就是一个技能，官方那 39 个是精选出来的，个个能打。

真正的玩法是：官方柜台挑好的直接拿，柜台外的自己淘。

经过几天的精挑细选，我最终留下15个skill，其中7个来自官方柜台，剩下8个是从社区淘来的。

接下来，小灰按照来源和分类，给大家逐一讲解这些技能。

## 二、官方内置三件套（1-3）

Codex 自带三个技能，很多人没注意，但它们是"技能的技能"，先认识它们，后面 12 个才能装起来。

1. skill-creator ⭐⭐⭐，这个我就不多说了，创建技能的技能，每个 Agent 必备的原始技能。

2. skill-installer ⭐⭐⭐，装技能的技能。先看看货架上有什么：

$skill-installer 看看有哪些技能

看中了哪个，一行命令进肚。

3. plugin-creator ⭐，把自用技能打包成插件分享给团队。

三个都在官方技能目录 openai/skills 的 .system 里，随 Codex 自动装好，无需下载（同目录还有 imagegen 和 openai-docs）。OpenAI 没先建货架，先给了你生产工具，态度很明显：技能这东西，攒比买重要。

## 三、开发流程八件套（4-11）

筛过的标准就一条：高频出现，装了确实比裸问 AI 靠谱。

4. create-plan ⭐⭐⭐，先规划后动手。

强制 Codex 在写第一行代码前产出实现计划，治"AI 上来就猛写、写完发现方向错了"。来自社区精选仓库 awesome-codex-skills。

地址：http://github.com/composio-community/awesome-codex-skills（create-plan 子目录）

5. grill-me ⭐⭐⭐，让 AI 反过来拷问你。

你提需求，它连环追问，每个分支聊清楚才放你走。多数人用 AI 的短板不是 AI 不行，是需求没想清楚，它把"想清楚"变成一场被迫完成的对话。

地址：http://github.com/mattpocock/skills

6. gh-fix-ci ⭐⭐⭐，CI 挂了别慌

官方精选区技能：排查并修复 GitHub Actions 上失败的 PR 检查。每周因为流水线挂掉浪费的半小时，交给它。

安装方式：$skill-installer gh-fix-ci。

7. gh-address-comments ⭐⭐，PR 评论批量清

官方精选区技能：处理当前分支 PR 上的 review 和 issue 评论，能改的直接改，要讨论的汇总成清单。审查意见十几条的 PR，一次清完。

安装方式：$skill-installer gh-address-comments。

8. tdd ⭐⭐，测试先行的老手艺。

红绿重构循环：先写失败的测试（红），写实现让它通过（绿），再重构。AI 猛写代码的冲动被测试摁住了。

地址：同 grill-me。

9. diagnosing-bugs ⭐⭐⭐，纪律化调试。

AI 修 bug 最大的毛病是"猜一个原因改一版试试"。这个技能定了纪律：先系统定位、收集证据，证据够了才动手。tdd 管让 bug 少生，它管 bug 生了怎么治。

地址：同 grill-me。

10. stop-slop ⭐⭐⭐，去 AI 腔。

清洗 AI 文本里的机翻味：delve、leverage、此外、值得注意的是，README 和提交信息是重灾区。一个专门治 AI 腔的技能活在 AI 编程工具里，多少有点黑色幽默。

地址：http://github.com/hardikpandya/stop-slop

11. sentry ⭐⭐，线上炸了先看这里。

官方精选区技能：通过 Sentry CLI 只读查询 issue，把生产错误总结成人话。排障第一步的定位工具，已经把服务上线的人值得装。

安装方式：$skill-installer sentry。

## 四、两个硬核补充（12-13）

前面八个管"把活干对"，这两个把别的世界接进来：一个把你读过的书接进来，一个把浏览器接进来。

12. book-to-skill ⭐⭐，把读过的书接进 Codex。

丢给它一本 PDF（你自己买的），它拆成结构化技能：核心心智模型、每章一个文件、一份速查表。章节按需加载，问到才读，官方实测 token 消耗只有直接塞整本 PDF 的 1/24 到 1/51。

地址：http://github.com/virgiliojr94/book-to-skill

安装方式：npx skills add virgiliojr94/book-to-skill。

13. playwright ⭐⭐⭐，把浏览器交给 Codex 开。

官方精选区技能：从终端自动化真实浏览器，填表、截图、抓数据，前端改完让它真机验证，抓竞品页面数据也靠它。

安装方式：$skill-installer playwright

想功能更全的话，可以访问社区版：http://github.com/lackeyjb/playwright-skill

社区版带有响应式检查和登录流程。

## 五、压轴的两个（14-15）

14. cangjie-skill ⭐⭐⭐，把读过的书变成装备。

元技能：喂它一本书、一个长视频或播客的文字稿，它跑流水线把里面的方法论拆成原子化能力卡，编译成可安装的技能包。你自己读完《穷查理宝典》记不住三条，它蒸馏出来的决策框架技能随时能被 Codex 调用。

它和刚才的第12个skill（book-to-skill）走了不同的路线：book-to-skill 管读书查询，它管方法论上岗干活。9.2k 星，中文作者出品。

地址：http://github.com/kangarooking/cangjie-skill

15. last30days ⭐⭐⭐，压轴。

主清单里唯一不写代码的技能。给它一个话题或一个人，它并行去搜 Reddit、X、YouTube、Hacker News、Polymarket，按真实互动和真金白银的下注排序，合成一份带引用的简报。作者有句话说得妙：Google 聚合的是编辑，last30days 搜索的是人。

地址：http://github.com/mvanhorn/last30days-skill

安装方式：npx skills add mvanhorn/last30days-skill -g

## 六、更重要的是：这些装备能带走

SKILL.md 是通用标准，Claude Code、Codex、WorkBuddy、Cursor 认的是同一份文件。实操就是复制粘贴：同一份技能文件夹，~/.claude/skills/ 和 ~/.codex/skills/ 各放一份，或者做个软链接。你在 Codex 里调教好的 tdd，Claude 那边零成本享用。

我的判断：工具会一直换，但攒下来的技能库是自己的。逛市场的心态是"这平台有什么我用什么"，攒装备的心态是"我的东西跟着我走"。 早期生态乱一点没关系，标准锁死了，资产就不会丢。

## 七、写在最后

总结一下用法：先拿内置三件套跑通安装流程；再按你的日常挑三四个流程技能（私心推荐 create-plan 和 stop-slop）；gh-fix-ci、gh-address-comments、sentry 三个能凑成一条发布链；两个硬核补充看需求装；last30days 记得带上。

15 个技能总览（⭐⭐⭐ 必装、⭐⭐ 推荐、⭐ 看痛点装）：

行动号召就一步：打开终端，输入 $skill-installer playwright，五分钟后你就知道我说的是什么了。

正在阅读这篇文章的朋友，你最常使用的 Codex 技能有哪些？欢迎在评论区聊一聊。

我是程序员小灰，欢迎大家关注我 @XiaohuiAI666，学习更多有用的AI玩法和副业经验。