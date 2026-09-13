# 如何通过中转站使用 Codex 等 Agent 工具

把大象塞进冰箱只要三步，装好并用上 Codex 其实也差不多：**下载应用 → 决定走官方还是中转站 → 配置好开始用**。这篇主要讲国内网络、或者没有 GPT Plus 的情况下，怎么通过中转站把 Codex 跑起来。

如果你已经有 GPT Plus / Pro，而且网络没问题，直接登录官方账号即可，不需要折腾中转站。至于中转站本身怎么挑、哪家质量更稳，可以看 [AI 中转站怎么选](/ai_coding/api_relay_station_selection.md)，也可以用 [Hvoy AI](https://www.hvoyai.com/) 查各家中转站的质量和评测，找一家适合自己的。

## 第一步：下载 Codex 应用

前往 Codex 官方网站（[https://openai.com/zh-Hans-CN/codex/](https://openai.com/zh-Hans-CN/codex/)）下载应用。Mac 和 Windows 都可以下，前提是有一个能访问的网络，这里不展开。Windows 可能会碰到 Microsoft Store 的网络问题，搜一下一般都有解决方案。

![下载 Codex 应用](assets/codex-tutorial/01-download-app.png)

## 第二步：两条路线，官方账号还是中转站

先看手里有没有 GPT Plus 或 Pro。

**路线一：有官方账号。** 如果你已经有 Plus / Pro，或者有渠道获取且预算充足，直接打开 Codex，选择登录账号，登录你的 GPT 账号就行。

![登录 GPT 账号](assets/codex-tutorial/02-login.png)

**路线二：走中转站。** 如果没有 Plus，或者充值困难，更推荐走中转站。所谓中转站，就是在你和 GPT 之间搭一座桥：你在中转站充值，把请求发给它，它用自己的号池把请求转给 GPT，帮你屏蔽掉中间那些复杂的环节。这部分会稍微繁琐一点，下面尽量讲得通俗些；如果还是卡住，可以直接问 AI，或者去找各家中转站的教程。

## 第三步：用 CC Switch 配置中转站

要接中转站，本来得手动改 Codex 的配置文件。好在现在有图形化工具帮忙，不用再去翻隐藏文件了。

前往 CC Switch 官网（[https://ccswitch.io/zh/](https://ccswitch.io/zh/)）下载这个软件，它可以帮助我们管理各个 Agent 的配置文件、接入不同的中转站。

![下载 CC Switch](assets/codex-tutorial/03-ccswitch.png)

打开 CC Switch，在最顶上选择 Codex 图标。如果你是第一次用，这时候应该只有一个官方配置，目标是调成像下面这样——有一个自己配置的中转站。

![CC Switch 配置页](assets/codex-tutorial/04-ccswitch-interface.png)

点击上方的添加按钮，进入添加页面，里面已经预置了不少还不错的中转站，这里以 Packy 为例。

![CC Switch 添加中转站](assets/codex-tutorial/05-add-provider.png)

点击 PackyCode，滑到下方可以看到它的官方网址（[https://www.packyapi.ai](https://www.packyapi.ai)）。

![选择 PackyCode](assets/codex-tutorial/06-select-packy.png)

点击访问官网，如果有语言问题，可以按页面提示切换成中文。

![Packy 官方网址](assets/codex-tutorial/07-packy-site.png)

然后注册账号。

![Packy 界面](assets/codex-tutorial/08-packy-interface.png)

接着来到钱包管理页面充值。Packy 要求至少充 50 元；如果你觉得贵，也可以换任意其他一家中转站，原则都是通的。充完之后要确保有余额，不然问 AI 也不会有响应。

![充值](assets/codex-tutorial/09-recharge.png)

![确认余额](assets/codex-tutorial/10-balance.png)

然后添加令牌。配置时选 codex 相关的分组（还会有很多其他分组，比如 Claude），创建完成后复制 token。**注意千万不要把它发给任何人**，万一被盗刷就麻烦了。token 就是密钥，有了它就能用你的账号向中转站发请求、换取 GPT 的回复，同时扣你账户的余额。

![添加令牌](assets/codex-tutorial/11-add-token.png)

![配置分组](assets/codex-tutorial/12-config-token.png)

![复制 token](assets/codex-tutorial/13-copy-token.png)

回到 CC Switch 的「添加 API Key」页面，把刚复制的密钥粘贴进去并保存。

![粘贴密钥](assets/codex-tutorial/14-paste-token.png)

然后点击配置文件右侧的连通测试按钮，看看能不能连上。其实这里省略了一步「配置 Base URL」，不过 CC Switch 已经帮我们预置好了。

![连通测试](assets/codex-tutorial/15-connection-test.png)

到这里不出意外的话，重新启动 Codex，就能像下面这样跟它对话了。

![开始对话](assets/codex-tutorial/16-run.png)

如果还有问题，可以查查各家中转站的文档。

![中转站文档](assets/codex-tutorial/17-docs.png)

## 几个提醒

- **中转站会变**：价格、稳定性、渠道质量都会变，别只认某一家。挑的时候可以参考 [AI 中转站怎么选](/ai_coding/api_relay_station_selection.md)，或者用 [Hvoy AI](https://www.hvoyai.com/) 看评测和排行。
- **token 别外泄**：它等同于你的钱包，泄露后可能被直接刷光额度。
- **先小额度试水**：第一次用建议少充一点，跑通流程、确认稳定之后再加大。
- **配置方式通用**：换成别家中转站，步骤基本一样，只是网址、token 和分组名称不同。

配置好之后，就可以在本地仓库里顺滑地用 Codex 干活了。多练几次，你会慢慢形成适合自己的提示模板和操作节奏。
