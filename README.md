# store · 私有市场数据仓库

这是**私密应用市场的数据落地处**：输入（`sources/`）、产物（`store/`）、以及全部 APK 二进制（Releases）。

- **现在公有**（开发期暂时如此）→ **部署期转私有**。
- 转私有正是 CF 遮蔽网关存在的理由，也是维护逻辑被拆出去的原因（GitHub **只对私有仓库**的 Actions 分钟数计费）。拆成了两个：公有的 [`forge`](../forge) 只放 workflow（**运行器**），逻辑的**实现**在私有的 [`forge-core`](../forge-core) 里 —— 所以"跑在公有仓库"和"逻辑不公开"能同时成立。
- 完整设计见 `market-spec/`（尤其 **03 · 维护逻辑**）。

> ⚠️ **这个仓库不执行任何维护逻辑**。维护数据那条链全靠**两个「转手就发车」的极薄 workflow**（`source-change` / `intake-incoming`，**一个功能一个文件**）把事件转告 `forge` —— 信标那一段是共用的复合动作 `.github/actions/beacon`；
> 另有一个 `publish-apk.yml`，它是**给"没有可读上游"的源复用的上传器** —— 只往 `_incoming` 里放文件，不读也不改 `sources/`（§4）。⚠️ 判据是**那个源的上游 `forge` 读不读得到**，不是"私有不私有"：同 `market-of-labs` owner 下的私有仓照样是**普通上游**，直接镜像即可（03 §8 / D55）。
>
> ⚠️ **对账（`reconcile`）在这一侧没有文件**（那个只做转发的 `reconcile.yml` 已于 2026-09-18 删除）。它的触发面是每日 cron，而 cron 与手动按钮都在 **`forge` 侧同一个文件**上（03 §4.4）—— 这边再放一个只会转发的空壳，就变成"一件事两个按钮"，而"今天到底跑了几次对账"要跨两个仓库去数。⚠️ 注意这和下面的 `intake-incoming` **不是一回事**：那个文件本来就必须存在（它接 `release` 事件），按钮是顺手加的 3 行；而 `reconcile.yml` 是**专门为了放一个按钮**才存在的一整个文件。

---

## 1. 目录

```
store/
├── sources/                    # 唯一事实源，一应用一文件
├── store/                      # 契约常量 + fdroid 工作区 —— 只有 forge 写
│   ├── endpoints.json          # 地址模板 —— **唯一一处"只改数据就改行为"的开关**
│   └── fdroid/
│       ├── config.yml          # 签名口令在这里，**绝不入库**（.gitignore 拦着）
│       ├── repo/               # fdroidserver 的工作目录，每轮重跑都重写，**不入库**
│       └── metadata/           # 每应用一份 <appId>.yml —— forge 每轮重写，**在 git 里**
├── repo/                       # 对外伺服的产物：**只有索引，没有 APK**（build-repo 每轮生成）
├── .github/
│   ├── actions/beacon/         # 发信标给 forge：不 checkout、不插值、只 POST 一次 dispatch
│   ├── workflows/source-change.yml   # 有人开了/改了申请单 → 通知 forge 处理那一张
│   ├── workflows/intake-incoming.yml # `_incoming` 队列被 Publish → 通知 forge 来搬
│   │                                 # 也是**手动按钮**所在（Run workflow，无入参）
│   ├── workflows/publish-apk.yml # 可复用：没有可读上游的源码仓把 APK 送进 _incoming（§4）
│   ├── dependabot.yml          # 每周升 beacon 里那个 action 的主版本
│   └── ISSUE_TEMPLATE/         # 维护数据源的两个入口
├── README.md
└── .gitignore
```

**`sources/` 是唯一事实源，产物是根 `repo/` 下那一份 F-Droid 仓库索引**
（`store/fdroid/metadata/` 是它的中间产物，也进 git —— 它逐份对应一个应用，比索引好读得多）。
`sources/{appId}.json` 里**人填的那半**（`name`/`author`/`upstream`…）与**forge 写的 `versions` 账本**
同住一个文件 —— 所以它也**整份由 forge 改写**，手改会在下一轮对账被盖掉
（版本账本曾经是独立的 `store/index.json`，已并入来源文件，D48）。

> **为什么产物在根 `repo/` 而不在 `store/fdroid/repo/` 里**：根 `repo/` 是对外**伺服**的那一份。
> `repoUrl` 的最后一段就是 `repo`（见 `store/endpoints.json`），客户端按 `repoUrl + "/" + 文件名`
> 取件。而 `store/fdroid/repo/` 是 fdroidserver 的工作目录，里面**索引与 APK 混在一起**；
> 往根目录拷的时候只带索引、不带 APK —— 每个应用的 APK 躺在**它自己的 Release** 里
> （`tag = {appId}`），由 CF 网关按文件名映射过去（04 契约）。
>
> ⚠️ **APK 不进 git**，`.gitignore` 里有 `/repo/**/*.apk` 兜底。它拦下的正是"有人顺手把 20MB
> 的 APK 提交进来"——而真发生了会很难看：APK 一旦进 git 历史就很难真的删掉。
>
> ⚠️ **`endpoints.json` 的 `repoUrl` 现在还是占位值**（`REPLACE-ME-BEFORE-PUBLISH.invalid`）。
> 它会被 `fdroid update` 写进 `index-v2.json` 的 `repo.address`，而客户端就是拿这个地址去取件的 ——
> 所以**真正发布前必须换成最终的 CF 地址**，否则所有人的客户端都会指向一个不存在的主机。
> 用 `.invalid` 是刻意的：它是 RFC 2606 的保留后缀，忘了改会**响亮地失败**，而不是悄悄指错地方。

---

## 2. 维护入口：开 issue

**数据源只由这两个模板（+ 手动触发的 action）维护，不靠手改文件** —— forge 落盘时会**整份重写**
`sources/{appId}.json`（含 `versions` 账本），手改的键在下一轮就被盖回去了。

### 落盘后长这样

标准源（上游是 GitHub Release）：

```json
{
  "id": "org.thirdparty.app",
  "name": "第三方公开示例",
  "author": "ThirdParty",
  "desc": "去广告的第三方客户端",
  "source": "github",
  "paused": false,
  "categories": ["工具"],
  "upstream": {
    "type": "github-release",
    "repo": "thirdparty/app",
    "assetPattern": "app-.*\\.apk$"
  },
  "abiWhitelist": ["arm64-v8a", "universal"]
}
```

> **`desc`（可选，≤20 字）会落成 F-Droid 的 `Summary`** —— 也就是客户端列表行里**应用名下面
> 那一行小字**。同一行上还有图标、名字、版本，`Summary` 是那行里唯一能写字的空白处，
> 所以才有这个字数上限（超了会被截断，并在回评里告诉你）。留空就只显示应用名。
> 它在文件里是独立字段，与 `name` 各走各的（曾经是拼进 `name` 的，D42 已作废）。

手动上传的 APK（没有 GitHub 上游）用 `source: "manual"`，二进制走 `_incoming` 暂存 Release：

```json
{
  "id": "com.you.closedapp",
  "name": "非自研闭源 APK",
  "author": "you(分发)",
  "source": "manual",
  "paused": false,
  "categories": ["私有"]
}
```

> **`id` 必须 = APK 包内真实的 `package`。** 它同时是文件名、git tag、和客户端安装时的身份，写错无法自动纠正。
> 两条入口都是这么取得的 —— **没有人手填 `id`**。

两个模板：

- **新增 · 标准源** → `.github/ISSUE_TEMPLATE/add-source.yml`
- **变更 · 移除** → `.github/ISSUE_TEMPLATE/change-source.yml`

变更单是**一拍**的：`forge` 校验后只改申请涉及的字段、回评、关单。被拒绝也会写明原因。

**新增单也是一拍的** —— 提交后一个 `issues` 事件把整条链跑完（**没有队列、没有标签**，见 03 §2.5.1）：

| 这一步 | 做什么 | 申请人看到什么 |
|---|---|---|
| 校验（纯本地） | 仓库形状、正则是否编译得过、勾选项是否在固定词表内 | —— |
| 探身份 | 去上游读出**包名**（APK 的 `package`）、**显示名**（APK 的 `label`）、**作者**（仓库 owner） | —— |
| 落盘 + 同步 | 写 `sources/{appId}.json` → 建 Release、镜像该版本 APK → 重建仓库产物（metadata + 索引）→ 回评核对 → **关单** | 一张表格列出读出来的身份 + 这一轮镜像进来的版本 |

**模板里没有 appId / 显示名 / 作者**，因为它们都**能自动取得**，而让申请人填一个会被覆盖的值只是在制造一条假回评。

> **失败会留开**：校验不过、探身份失败、落盘失败，`forge` 都**回评说明原因并让单开着** ——
> 单开着就是"这件事还没成"。要重跑，**编辑正文**即可（同一个事件会重新发车），不必重开单。

> ⚠️ **这条路的代价：仓库填错会静默收错应用。** 没有人再手工核对"你要的是哪个包"，
> 于是**多包名是唯一剩下的把关点** —— 一个 Release 里若出现两个不同 `package`，`forge`
> **不猜**，直接把两个名字都回评出来、请申请人用「资产匹配正则」消歧（改正文重填即可，不用重开单）。
>
> 另一处不再设防的是：申请人勾的分类/ABI 由 `validateAdd` 对固定词表校验，模板的选项与
> Go 侧的词表由 `issue_test.go` 的 `TestCheckboxVocabularyMatchesGo` 钉死，两边分叉会当场炸。

**单自己就是那条记录**：不建数据库、不打标签，**单还开着**就是"这件事还没成"。
想改内容就**编辑正文**，不必重开单（编辑后的正文走的是**同一套**解析规则，见 `internal/job/intake.go`）。
**收尾之后的失败不需要重试** —— `sources/` 里的文件已经在了，每日对账是幂等的，缺哪补哪。

> **手动上传没有新增单**（D52）。它不经 `add-source.yml`：那张单里根本没有能表达
> "我没有上游仓库"的字段（`repo` 是必填）。条目由 §4 那条搬运链**按 APK 内容当场建**：
> `id` = 包名、`name` = APK 的 `label`、`author` 先记 **`未知`**。
> 所以"传一个 APK"就是收录一个新来源的全部动作，传完它就在索引里了。
>
> 代价是**作者只能先占位**：APK 里没有作者字段，也没有上游仓库可以取 owner。而 `author`
> 是必填（空着会让**整批来源**判失败，不是只废掉这一条），所以 `未知` 是唯一能填的值 ——
> 它也是**唯一**的提示信号：看到它就说明这条来源还在等一张 `change-source.yml`。

---

## 3. 停更 vs 移除（**最容易搞错的一处**）

| 你想做的 | 正确做法 | 结果 |
|---|---|---|
| **不再追新版本** | 把 `sources/{appId}.json` 的 `paused` 改成 `true` | 它的 metadata 被清掉 ⇒ **不在索引里**；设备端安静（不会提示更新），但**新设备也装不到它了** |
| **彻底不再提供** | 删掉 `sources/{appId}.json` | 同上，只是**账本一起没了**（`paused` 是可逆的，删文件不是） |

> ⚠️ **这一节与规格还没对齐，先按上表理解。** `market-spec` 02 §2.9.2 说的是 `paused` ⇒
> 「条目**留在索引里**、不再有新版本」，也就是"老设备安静、新设备照样能装"；而本仓的实现是
> **不渲染它的 metadata**（依据是 02 §2.5 的「`paused` 不生成任何东西」），于是它**整个从
> 索引里消失**。两句规格话不可能同时成立 —— F-Droid 侧要做到 §2.9.2 那个效果，得**同时保留
> metadata 与 cache 里的老 APK**，只是不再去上游取新版。**这条分歧还没解决**（02 §2.5 与
> §2.9.2 互相矛盾），等真机验证过客户端行为再定。

**结论：想停更就用 `paused`，别删文件。** 理由不是"设备端行为更好"（那条现在我们说了不算，
见上），而是**可逆**：`paused` 之后账本原封不动，把 `true` 改回 `false` 就能恢复；
删文件则是把"已经镜像过哪些版本"一起丢了，而那份账本只能从 Release 现状重建。

---

## 4. 手动上传 APK 的流程（`source: "manual"`）

1. 打开 Releases → `_incoming` → 上传 APK。**一次可以传多个**，
   属于不同 App、不同版本、不同 ABI 的都行 —— 下面第 3 步会把**整批**一次搬完。
2. **点 "Publish release"** —— 就是这一下发车（`release: published` 事件接住它，D57）。
3. `forge` 解析每个 APK 的 `package / versionName / versionCode / ABI` → 改名搬运进对应 `{appId}` 的正式 Release → 重建仓库产物。
4. `forge` 把 `_incoming` **改回 draft**、删掉那个 tag 引用，并删掉已搬走的 asset（清场）。

> **`sources/` 里还没有这个包名时，条目就在第 3 步当场建出来**（D52）：包名 = `id`、
> APK 的 `label` = 显示名、`author` 先记 `未知`。所以新建一个手动来源**不需要任何单子** ——
> 传一次 APK 就够了，传完它已经在索引里。作者与简介之后用 `change-source.yml` 补。

> ⚠️ **为什么非要点 Publish 不可**：往 Release 上传 / 改名 / 删 asset **不触发任何 `release`
> 事件**（GitHub 的硬限制，03 §3.3）—— 光把文件传上去，什么都不会发生。所以**发车必须挂在另一个
> 动作上**，而"发布"正是你本来就期待的那个收尾动作（传完文件总要留下点什么吧）。
>
> ⚠️ **没反应时点一次按钮**：本仓库 Actions → **`搬运队列·转发`** → Run workflow（没有入参）。
> 这是唯一需要的排查动作 —— 它跟 Publish 走的是同一条下游，效果一样。
> 按钮在本仓库而不是只在 `forge`，是因为你刚在这个仓库的 Release 页面上传完文件，省一次仓库切换。
>
> ⚠️ **一个功能一个入口文件。** 这个仓库的 Actions 页面上有**两个**入口，各自只干自己那一件事 —— **互不串台**。
> ⚠️ **界面显示中文名、文档一律用文件名**（路由靠的是文件名，不是这个名字 —— 它只是给人看的），对照如下：
>
> | 界面显示 | 文件 | 触发面 | 干的事 |
> |---|---|---|---|
> | **`搬运队列·转发`** | `intake-incoming.yml` | `release: published` + **手动按钮** | 通知 `forge` 搬 `_incoming`（也是 Publish 那条路的下游）。**唯一带按钮的那个** |
> | **`申请单·转发`** | `source-change.yml` | `issues: [opened, edited]` | 有人开/改了申请单时通知 `forge` 处理**那一张**。**没有按钮** |
>
> ⚠️ **`申请单·转发` 这张单子的"重跑"不是点按钮**：本仓库这一侧只有开/改 issue 两条触发面。
> 要重跑某一张申请单，**编辑那张 issue**（改一下再改回来即可），或者去 `forge` 点它的
> **`处理申请单`** 按钮（那边带 `-issue N` 入参）。这是本文件里一处订正 —— 从前这里写着
> "重跑某一张申请单"，但那个按钮在本仓库并不存在。
>
> ⚠️ **对账（`reconcile`）在Actions 页面上没有入口**（那个只做转发的 `reconcile.yml` 已于 2026-09-18 删除）。
> 它的三个触发面 —— 每日 cron、手动按钮、`repository_dispatch` —— **都在 `forge` 侧同一个文件上**（03 §4.4）：
> 一条链只有一个发车点，"今天到底跑了几次对账"才不用跨两个仓库去数。
> ⚠️ 这**不妨碍**上面那张表里的 `intake-incoming.yml` 保留按钮：那个文件本来就必须存在（它接 `release` 事件），
> 按钮是顺手加的 3 行；而删掉的那个是**专门为了放一个按钮**才存在的一整个文件。
>
> 两个**都只是把话转告 `forge`**（`beacon` 复合动作为此存在）—— 名字里那个"转发"就是这件事：
> 本仓库不执行任何维护逻辑。`forge` 侧的同名文件才是**真正动手**的那一端（它的名字没有"转发"二字）。
> 从前它们挤在一个 `forward.yml` 里、靠一个 `verb` 下拉框区分，而那个下拉框是可以选错的。
>
> ⚠️ **搬不进去的会留在队列里**，逐条写明原因、**不静默丢弃**（03 §3.2）；看一眼 forge 那次
> run 的日志就知道哪几条还在等人工。重跑一次搬运即可，幂等（已搬成的会被跳过）。
>
> ⚠️ **清场用「改回 draft」，绝不用「删 Release」**：删 Release 会让 tag 消失；若该仓库曾开启过
> Immutable Releases，该 tag 会被**永久烧毁**且无法重建。
>
> ⚠️ 第 4 步删的是 **tag 引用**（`refs/tags/_incoming`），**不是 Release** —— 两件事完全不同，别
> 看混。那一步必须做：`release` 事件跑的是 **tag 所指提交**上那份
> `.github/workflows/intake-incoming.yml`，而引用一旦建立就**不再移动**，于是本仓库默认分支上怎么
> 改触发器、队列这条路都看不见（症状：Publish 了却什么都没发生，而 Actions 页面干干净净）。
> 删掉之后，下一次 Publish 会在**当时的**默认分支 HEAD 上重建它（03 §3.2）。

### 自研 App：让源码仓的 CI 自动走这四步

⚠️ **先看判据**：`forge` 读得到上游 ⇒ **什么都不用做**，它和别的应用一样在每日对账里被自动镜像。
**同 `market-of-labs` owner 下的私有仓就属于这一类** —— `forge` 的 GitHub 客户端本来就带那把 PAT，
在 PAT 的仓库清单里加上它（Contents: **Read**）即可，零代码改动（03 §8 / D55）。

真正需要下面这套的是**上游 `forge` 够不着**的源：源码仓在**别的 owner** 名下（fine-grained PAT 只覆盖一个 owner），
或者根本没有上游（`source: "manual"`）。它们走的是**同一条**上传队列，只是把上面 4 步自动化 ——
本仓库的 `publish-apk.yml` 是可复用 workflow，App 仓构建完调一下即可：

```yaml
jobs:
  build:                      # 构建 job 与平时一样，只多一步
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: ./gradlew assembleRelease
      - uses: actions/upload-artifact@v7
        with: { name: apk, path: app/**/*.apk }   # 名字必须是 apk
  publish:
    needs: build
    uses: market-of-labs/store/.github/workflows/publish-apk.yml@master   # 本仓库默认分支是 master（不是 main）
    # 没有 with: —— 落点由 APK 内容决定（D52），这个 workflow 不需要知道 appId
    secrets: { store-token: ${{ secrets.STORE_TOKEN }} }   # 对 store 有 contents:write 的细粒度 PAT
```

它做两件事：把 APK 传进 `_incoming`、**发一个 `repository_dispatch` 叫 `forge` 来搬** ——
之后与手动路径完全一样（第 3、4 步）。

> ⚠️ **这里曾经有个 `with: { app-id: ... }`（2026-09-16 删）**：唯一用途是在调用方的日志里对一下
> 包名收没收录，**对结果零影响**（打错了也只多一条作者为「未知」的来源，而那个会自愈）。它却让
> 每个调用方都得传一个被忽略的值，传错还会直接报错（可复用 workflow 收到未声明的 input 会失败）。

> ⚠️ **它拿的是对 `store` 与 `forge` 都有 `contents:write` 的 PAT**：那声信标是往 `forge` 仓库发的，
> 所以同一把 token 也得够得着那边。调用方在自己的仓库里放这把 token 时要知道这一点。

> ⚠️ **它不碰队列的 draft 状态** —— 那个"上传前先把队列复位成 draft"的步骤同一天（2026-09-16）删了。
> 它当年是**反的**：`gh api -f draft=true` 传的是字符串 `"true"`，GitHub 只当 `draft=false`，于是每次
> CI 上传前先把队列**发布掉**、还顺手把 tag 名降级成 `untagged-<sha>`，下一次搬运就认不出队列了
> （症状正是"传了却没反应"）。布尔值得用 `-F`。而它想防的那件事今天也不用防：队列的状态由 `forge`
> 清场维持（第 4 步）。

> ⚠️ **App 仓必须在同一 org 才调得动**（store 转私有后，私有仓的可复用 workflow 只对同 org 开放）。
> 否则就把那个文件里的两步 `gh api` 就地抄进 App 仓的 workflow —— 同样只需要那把 PAT，不需要 store 公开。
> ⚠️ **这两条约束同源，别当成两个独立条件**：一个仓不在我们的 org 里，就既调不动可复用 workflow、`forge` 的 PAT 也读不到它的 Release ——
> 所以"得走队列"和"得同 org"说的其实是同一件事（fine-grained PAT 只覆盖一个 owner，03 §8）。

---

## 5. 地址配置：`store/endpoints.json`

```json
{
  "tagTemplate": "{appId}",
  "assetNameTemplate": "{appId}-{version}-{abi}.apk",
  "repoUrl": "https://REPLACE-ME-BEFORE-PUBLISH.invalid/fdroid/repo"
}
```

前两个是**契约常量**，不是配置：

- `tagTemplate` = `{appId}` —— "一应用一 Release"的依据，也是 CF 网关从文件名**反查出该去哪个 Release 取件**的依据（04 §3.2）；
- `assetNameTemplate` = `{appId}-{version}-{abi}.apk` —— 全部历史 asset 的命名，`{appId}` 不含 `-` 正是"按第一个 `-` 切段"能成立的前提（02 §2.9.1）。

改它们 = 改 Release 结构与每一条 asset 的文件名，要配套做数据迁移。**`repoUrl` 才是配置**：

- 客户端按 `repoUrl + "/" + 文件名` 取件，而这个前缀由 `fdroid update` **写死在索引里**（`index-v2.json` 的 `repo.address`）—— 所以改它只需改这一行 + 重跑一次 `build-repo`，**客户端不用动**；
- 现在它还是**占位值**。`.invalid` 是 RFC 2606 的保留后缀，忘了改会**响亮地失败**，而不是悄悄把客户端指向一台不存在的主机；
- 本机验证时要**临时**把它改成那个本地 HTTP 服务的地址，验完改回来（03 §7 #11 那条真机实测就是这么做的）。

> CF 网关接进来之后这一行**仍然是唯一的开关** —— 网关只负责把 `/fdroid/repo/<文件名>` 落到正确的地方（索引读本仓，APK 读各应用自己的 Release），地址形状不变。

---

## 6. 当前状态

- `sources/` **12 条**来源，`store/fdroid/metadata/` **12 份** `<appId>.yml`，一一对应
  —— 后者由 `forge-core` 的黄金测试与实际渲染结果**逐字节**比对（`internal/store/golden_test.go`）。
- **根 `repo/` 还不存在**：索引要等 `build-repo` 在 CI 里跑过第一轮才会生成。
  在那之前客户端拉取是 404 —— **这是预期，不是故障**。
- ⚠️ **一条死来源**：`com.obtainium.companion`（"私有市场同步器"）的上游是
  `market-of-labs/companion`，而那个仓库随 D58 一起删了。于是它现在渲染出的
  `SourceCode` / `IssueTracker` 是**两个 404 链接**，下一轮对账在取件那一步也会空手而归。
  要么给它换个上游，要么按 §3 处理掉。

---

## 7. 部署前置检查

| # | 检查项 |
|---|---|
| **1** | **本仓库及其所属 org 的「Immutable releases」必须为 OFF**（2025-10-28 GA）。开启后 asset 不能增删改、tag 不能删/移 —— 本设计的追加上传与 `_incoming` 清场全部失效；删除不可变 Release 会**永久烧毁该 tag**，而 `tag = {appId}` 不可重建。**"先开后关"也不安全**（已固化的 Release 不受影响）。 |
| 2 | `store` 转私有后按私有仓库计费 —— 逻辑跑在公有的 `forge`（其实现来自私有的 `forge-core`），本仓库只跑一个 dispatch 步骤。⚠️ **这是这套拆分的全部意义**：下载 APK、跑 `fdroid update` 这些重活全在 `forge` 上，本仓库这边只有一次转发 |
| 3 | 本仓库与 `forge` 各自都能解析出 **同名** secret `GH_PAT`（**仓库级或组织级都行** —— 现状是本仓库用仓库级、`forge` 走组织级），值是**同一把** fine-grained PAT，仓库范围必须含 `store`(Contents RW + Issues RW) / `forge`(Contents RW) / **`forge-core`(Contents R)**（最后一个：`forge` 要下载它的 Release 里的执行体；转私有后也是靠这把 PAT 读，D40） |
| 3b | `forge-core` 必须**已发布过至少一个带 `forge-linux-amd64` asset 的 tag** —— 公有的 `forge` 浮动取 latest，一个 Release 都没有时取件直接失败 |
| 4 | 本仓库 workflow 保持 `permissions: {}` |
| 5 | 确认 `_incoming` **存在**（draft 或 published 都行）。⚠️ **别再手工把它 Convert to draft**：从 D57 起，"上传后 Publish" 是正常动作，published 是上传到清场之间那个**正常的**中间状态，`forge` 清场会自己拧回去（§4 第 4 步）。它若长停在 published，那只有一个意思 —— 搬运没跑成，点一次手动按钮（`intake-incoming` 那个 workflow）即可 |
