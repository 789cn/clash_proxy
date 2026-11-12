# surge_rules

## [`复刻-自用 Surge 规则集：Steam（国区商店/登录直连 + 社区/创意工坊走代理）、坦克世界国际服。域名经 DoH 双视角 DNS 实测核实。`](https://github.com/mrblack0407/surge_rules)

自用的 Surge 规则集。每条域名都经 DNS 实测核实，不抄第三方规则集；核实方法与原始数据在 [`docs/`](docs/) 下按主题分文件记录。

| 规则集 | 内容 | 建议策略 |
| --- | --- | --- |
| [`rules/Steam-Community.list`](https://raw.githubusercontent.com/mrblack0407/surge_rules/refs/heads/main/rules/Steam-Community.list) | Steam 社区、创意工坊内容、聊天、直播、Valve 官网 | 你的代理策略组 |
| [`rules/Steam-Direct.list`](https://raw.githubusercontent.com/mrblack0407/surge_rules/refs/heads/main/rules/Steam-Direct.list) | Steam 国区商店、账号登录、客户端 API、下载 CDN | `DIRECT` |


完整接入示例见 [examples/surge-snippet.conf](https://github.com/mrblack0407/surge_rules/blob/main/examples/surge-snippet.conf)。

---

# Steam

## 为什么要拆成两份

Steam 在国内是**一半能直连、一半不能**的服务，但常见规则集都把它当成一个整体：

- blackmatrix7 的 `Steam.list` 用一条 `DOMAIN-SUFFIX,steamstatic.com` 覆盖全部静态资源。可 `store.cloudflare.steamstatic.com`（商店）与 `community.cloudflare.steamstatic.com`（社区）落在不同的 Akamai 边缘配置上，一条规则把两者绑成同一策略，无论选直连还是选代理都有一半是错的。
- 它的 `SteamCN.list` 把 `steamusercontent.com` 判为 `DIRECT`。那是 Valve 的 UGC 云存储域（Akamai），承载旧式创意工坊项的文件本体、预览图与云存档，属于社区侧内容，却和社区其余资源被分到了相反的策略。
- `SteamCN.list` 里还有 7 条完美世界 CS:GO / Dota2 国服的遗留域名早已 NXDOMAIN（清单见 [docs/steam-verification.md](docs/steam-verification.md)）。

所以这里按**功能面**拆成两份：能直连且必须直连的走 `Steam-Direct.list`，国内用不了的走 `Steam-Community.list`。

## 用法

```
Steam = select, 节点选择, DIRECT
```

```
# ⚠️ 社区表必须排在直连表之前
RULE-SET,https://raw.githubusercontent.com/mrblack0407/surge_rules/main/rules/Steam-Community.list,Steam
RULE-SET,https://raw.githubusercontent.com/mrblack0407/surge_rules/main/rules/Steam-Direct.list,DIRECT
```

两份列表的域名互不重叠，但社区表放前面可以保证以后扩充直连表时不会意外吞掉社区域名。

如果你原本引用了 blackmatrix7 的 `SteamCN.list` 或 `Steam.list`，**把那两行删掉**再加上面两行，否则 `steamusercontent.com` 会被上游的 `DIRECT` 规则抢先匹配。

## 拆分依据

走 **直连** 的是商店、登录、支付、客户端 API 与游戏下载。除了速度，更重要的原因是 **Steam 按出口 IP 判定账号地区**：这些域走境外节点，会让国区价格和人民币支付方式消失，下载还会从国内 CDN 掉到境外源站。

走 **代理** 的是 `steamcommunity.com` 及其静态资源、创意工坊内容、聊天、直播。这是 Steam 唯一需要翻墙的主要功能面，登录本身不在其中 —— 客户端登录走 `api.steampowered.com`，手机令牌也走同一个域，都在直连侧。

2026-09-21 从中国电信 AS4134 实测：`steamcommunity.com` 直连 0/15 全败、经节点 9/9 成功；而社区静态资源与 UGC 域（`community.*.steamstatic.com`、`steamusercontent.com` 等）直连 6/6 可用。也就是说**创意工坊「订阅」离不开代理，「下载」其实不需要** —— 但 UGC 直连实测也并不更快（101–127 KB/s vs 经节点 143–355 KB/s），所以整个社区面仍按同一策略走代理。数据见 [docs/steam-verification.md]([docs/steam-verification.md](https://github.com/mrblack0407/surge_rules/blob/main/docs/steam-verification.md))。

具体到症状：

| 症状 | 缺的规则 |
| --- | --- |
| 商店显示美元价、没有微信 / 支付宝 | `steampowered.com` 走了代理 |
| 下载速度只有几百 KB/s | `steamcontent.com` 或国区 CDN 走了代理 |
| 社区页打得开但头像图片全裂 | 少了 `community.*` / `avatars.*.steamstatic.com` |
| 创意工坊页面打不开 / 无法订阅 | 少了 `steamcommunity.com` |
| 创意工坊预览图裂、旧式 mod 下不动 | `steamusercontent.com` 被判成了直连 |
| 好友列表 / 聊天连不上 | 少了 `steamcommunity.com`、`steam-chat.com` |

## 不收录什么

- **第三方查询站**（SteamDB、steamstat.us）—— 不是 Valve 官方域。要代理的话自己加一条 `DOMAIN-SUFFIX,steamdb.info`。
- **第三方商店**（Humble Bundle、Fanatical）与盗版站 —— 上游 `Steam.list` 收了，这里不收。
- **境外 ISP 本地缓存节点**（澳洲 on.net、新西兰 orcon / slingshot、俄罗斯 webra、Comcast 等）—— 对国内用户没有意义。
- **已停运的 Valve 游戏域**（playartifact.com、underlords.com）。

---

# 坦克世界（World of Tanks）

国际服规则集，附国服 / 俄服排除表。核实方法与原始数据见 [docs/wargaming-verification.md](docs/wargaming-verification.md)。

## 用法

在 `[Rule]` 小节加入以下两行，**排除表必须在前**：

```
RULE-SET,https://raw.githubusercontent.com/mrblack0407/surge_rules/main/rules/Wargaming-CN-RU-Exclude.list,DIRECT
RULE-SET,https://raw.githubusercontent.com/mrblack0407/surge_rules/main/rules/WorldOfTanks.list,Wargaming
```

其中 `Wargaming` 是一个策略组，在 `[Proxy Group]` 里定义：

```
Wargaming = select, DIRECT, 节点选择, icon-url=https://cdn.jsdelivr.net/gh/mrblack0407/surge-icons@main/icons/Wargaming.png
```

选节点时按你玩的大区来：欧服选欧洲节点，美服选美西节点，亚服选新加坡或日本节点。绕远的节点只会让延迟更差。

## 覆盖范围

`WorldOfTanks.list` 用 `DOMAIN-SUFFIX` 覆盖以下域名及其全部子域：

- **登录与对局后端** — `worldoftanks.net`、`worldoftanks.eu`、`worldoftanks.com`、`worldoftanks.asia`
- **账号 / 门户 / 开放 API** — `wargaming.net`、`wargaming.com`、`wg.net`
- **客户端下载与 CDN** — `wgcdn.co`
- **官方周边** — `wotreplays.eu`、`worldoftanks.co.uk`、`worldoftanks.ru`

外加一条精确主机 `DOMAIN,wds.gcdn.co` —— WGC 下载通道 WDS 在 G-Core CDN 上的节点，与 `wds.wargaming.net` 同指一个资源。`gcdn.co` 是 G-Core 的公共 CDN 域，故只收这一个主机名，不加后缀。

## 已知局限：对局中的实时流量

**这一点决定了这套规则能帮你什么、不能帮你什么，请务必读完。**

坦克世界客户端登录后，由登录服务器下发对局节点的 **IP 地址**，客户端随后直接以 UDP 连上去（官方文档列出的 UDP 端口含 12000–29999）。这一步**不经过 DNS 解析**，因此域名规则匹配不到它。

也就是说，这套规则集能可靠覆盖的是：

- ✅ 游戏中心（WGC）登录、客户端更新与补丁下载
- ✅ 账号登录、车库、商城、任务与活动页
- ✅ 官网、开放 API、录像站

不保证覆盖的是：

- ❌ 进入战斗后的实时对局流量

如果你的问题是**登录慢、更新慢、网页打不开**，这套规则集对症。如果问题是**对局中掉线或高延迟**，域名规则帮不上，需要额外的 IP-CIDR 规则。

仓库没有内置 IP 段，是有意为之：国际服后端跨 G-Core Labs（AS199524）与 AWS 两套设施，地址随时轮换。硬编码的 IP 段要么很快过期，要么写宽到会误伤同机房的其他服务——两种都比没有更糟。真要做，建议自己抓一次对局流量取实际地址，针对性地加几条。

## 不收录什么

- **第三方生态站点**（wotinspector、tomato.gg、wot-life、XVM、wgmods 等）—— 与官方服务无关，且各自的 CDN 归属独立，混进来会让规则集难以维护。
- **其他 Wargaming 游戏**（战舰世界、战机世界、闪击战）—— 域名体系各自独立，本仓库只做坦克世界。

---

## 更新

域名有变动时欢迎提 Issue 或 PR。提交新域名请附上核实依据，标准见 [docs/steam-verification.md](docs/steam-verification.md) 与 [docs/wargaming-verification.md](docs/wargaming-verification.md)。

## License

[MIT](LICENSE)
