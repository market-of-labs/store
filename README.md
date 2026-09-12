# store · 私有市场数据仓库

这是**私密应用市场的数据落地处**：输入（`sources/`）、产物（`store/`）、以及全部 APK 二进制（Releases）。

- **现在公有**（为了让开发期的 raw 直链可直接用）→ **部署期转私有**。
- 转私有正是 CF 遮蔽网关存在的理由，也是维护逻辑被拆出去的原因（GitHub **只对私有仓库**的 Actions 分钟数计费）。拆成了两个：公有的 [`forge`](../forge) 只放 workflow（**运行器**），逻辑的**实现**在私有的 [`forge-core`](../forge-core) 里 —— 所以"跑在公有仓库"和"逻辑不公开"能同时成立。
- 完整设计见 `market-spec/`（尤其 **03 · 维护逻辑**）。

> ⚠️ **这个仓库不执行任何维护逻辑**。维护数据那条链全靠一个「转手就发车」的极薄 workflow 把事件转告 `forge`；
> 另有一个 `publish-apk.yml`，它是**给私有源码仓复用的上传器** —— 只往 `_incoming` 里放文件，不读也不改 `sources/`（§4）。

---

## 1. 目录

```
store/
├── sources/                    # 唯一事实源，一应用一文件（**现在是空的**，只有 .gitkeep）
├── apps.json                   # 最终清单 —— 放**根目录**，因为客户端唯一消费的就是它（首条落地前不存在）
├── store/                      # 其余产物 + 契约常量 —— 只有 forge 写
│   └── endpoints.json          # 地址模板（第一期 ⇄ 部署期的唯一开关）
├── .github/
│   ├── workflows/forward.yml   # 薄转发：不 checkout、不插值、只 POST 一次 dispatch
│   ├── workflows/publish-apk.yml # 可复用：私有源码仓的 CI 把 APK 送进 _incoming（§4）
│   ├── dependabot.yml          # 每周升 forward.yml 里那个 action 的主版本
│   └── ISSUE_TEMPLATE/         # 维护数据源的两个入口
├── README.md
└── .gitignore
```

**`sources/` 是唯一事实源，产物是 `apps.json` 与 `store/endpoints.json`。**
`sources/{appId}.json` 里**人填的那半**（`name`/`author`/`upstream`…）与**forge 写的 `versions` 账本**
同住一个文件 —— 所以它也**整份由 forge 改写**，手改会在下一轮对账被盖掉
（版本账本曾经是独立的 `store/index.json`，已并入来源文件，D48）。

> **为什么清单在根目录而不在 `store/` 里**：`apps.json` 是**唯一直接伺服给客户端**的文件，
> 它的路径就是设备的默认清单地址（`raw.githubusercontent.com/market-of-labs/store/master/apps.json`）。
> 把它放在根目录，地址里就不会出现 `store/store` 这种「仓库名与目录名重复」的段；
> `endpoints.json` 是给 forge 与 CF 读的契约常量，不需要短路径，留在 `store/` 里。
>
> ⚠️ **路径里的分支名是 `master`**（本仓库的默认分支），别顺手写成 `main`。
> `companion` 仓库用的是 `main`，两者**不同名是有意的** —— 对齐方式是改地址，不是改默认分支。
> 所以任何硬编码这条 raw 直链的地方（`companion/app/build.gradle.kts` 的内置默认值、
> 上述两处文档）都必须同时改，漏一处就是设备端一个 404。

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

> **`desc`（可选，≤20 字）是列表里唯一能写字的地方。** Obtainium 的列表行只有
> 「图标 + 名字 + `by 作者` + 版本」，没有描述字段（`about` 那种长文本只在应用**详情页**里）。
> 所以 `desc` 由 `forge` 在合成清单时**拼进 `name`**，设备上显示成 `第三方公开示例 · 去广告的第三方客户端`。
> 它在文件里是独立字段 —— 想改分隔符或改简介，都不用动别的条目。

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

> **`id` 必须 = APK 包内真实的 `package`。** 它同时是文件名、git tag、和 Obtainium 的安装身份，写错无法自动纠正。
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
| 落盘 + 同步 | 写 `sources/{appId}.json` → 建 Release、镜像该版本 APK → 重建 `apps.json` → 回评核对 → **关单** | 一张表格列出读出来的身份 + 这一轮镜像进来的版本 |

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
> 所以"传一个 APK"就是收录一个新来源的全部动作，传完它就在 `apps.json` 里了。
>
> 代价是**作者只能先占位**：APK 里没有作者字段，也没有上游仓库可以取 owner。而 `author`
> 是必填（空着会让**整份清单**判失败，不是只废掉这一条），所以 `未知` 是唯一能填的值 ——
> 它也是**唯一**的提示信号：看到它就说明这条来源还在等一张 `change-source.yml`。

---

## 3. 停更 vs 移除（**最容易搞错的一处**）

| 你想做的 | 正确做法 | 结果 |
|---|---|---|
| **暂时/永久不再更新** | 把 `sources/{appId}.json` 的 `paused` 改成 `true` | 条目留在清单里、`latestVersion` 冻结；设备端表现为「**无更新**」（安静） |
| **彻底不再提供** | 删掉 `sources/{appId}.json` | 新设备不再收录；**但已同步设备上那一行不会消失**，需用户手删 |

**为什么移除不传播**：deep link 的 `import()` 只有创建/覆盖，**没有删除动作**。清单里没有某条 ≠ 让 Obtainium 删掉它。这是路线 C（零改动 Obtainium）的结构性代价，客户端无法补救（读不到 Obtainium 的行数据）。

> **结论：想停更就用 `paused`，别删文件。**

---

## 4. 手动上传 APK 的流程（`source: "manual"`）

1. 打开 Releases → `_incoming`（**常驻 draft**）→ 上传 APK。
2. 点 **Publish release**。
3. `release: published` → `forward.yml` → `forge` 解析每个 APK 的 `package / versionName / versionCode / ABI` → 改名搬运进对应 `{appId}` 的正式 Release → 重建清单。
4. `forge` 把 `_incoming` **改回 draft** 并删掉已搬走的 asset（清场）。

> **`sources/` 里还没有这个包名时，条目就在第 3 步当场建出来**（D52）：包名 = `id`、
> APK 的 `label` = 显示名、`author` 先记 `未知`。所以新建一个手动来源**不需要任何单子** ——
> 传一次 APK 就够了，传完它已经在 `apps.json` 里。作者与简介之后用 `change-source.yml` 补。

> ⚠️ **为什么必须有「点 Publish」这一步**：往 Release 上传/改名/删 asset **不触发任何 `release` 事件**。Publish 是唯一能让流程发车的动作。
>
> ⚠️ **清场用「改回 draft」，绝不用「删 Release」**：删 Release 会让 tag 消失；若该仓库曾开启过 Immutable Releases，该 tag 会被**永久烧毁**且无法重建。

### 自研 App：让源码仓的 CI 自动走这四步

自研 App 的源码仓是私有的，`source: "github"` 那条路走不通（forge 匿名读不了私有仓，我们也不打算给它上游读凭据）。
它走的是**同一条**上传队列，只是把上面 4 步自动化 —— 本仓库的 `publish-apk.yml` 是可复用 workflow，App 仓构建完调一下即可：

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
    with: { app-id: com.you.closedapp }
    secrets: { store-token: ${{ secrets.STORE_TOKEN }} }   # 对 store 有 contents:write 的细粒度 PAT
```

它做三件事：**先确认 `sources/<appId>.json` 已在案**（挡住打错的 `app-id` —— 真正的落点由 APK 内容决定，
所以这个 app-id 对不上时，写错的后果是 store 里悄悄多出一条陌生来源）、
把 APK 传进 `_incoming`、**Publish** —— 之后与手动路径完全一样（第 3、4 步）。

> ⚠️ **上传前它会把队列复位成 draft**：发车靠 `published` 事件，而「已经是 published 再 PATCH `draft=false`」
> **没有状态跃迁**，事件不会发（资产会静静躺在队列里，谁也不会来搬）。这条复位同时是**恢复手段** ——
> 上一轮搬运失败（队列停在 published、还带着残骸）或 `published` 事件半路丢了，重跑一次 CI 就重新走一遍
> draft → published 的真实跃迁，不需要人去网页上手工 Convert to draft。

> ⚠️ **App 仓必须在同一 org 才调得动**（store 转私有后，私有仓的可复用 workflow 只对同 org 开放）。
> 否则就把那个文件里的两步 `gh api` 就地抄进 App 仓的 workflow —— 同样只需要那把 PAT，不需要 store 公开。

---

## 5. 第一期 ⇄ 部署期：只差 `store/endpoints.json` 一行

| | 第一期（现在，无 CF） | 部署期 |
|---|---|---|
| `assetUrlTemplate` | `https://github.com/market-of-labs/store/releases/download/{appId}/{fileName}` | `https://<cf域>/asset/{appId}/{version}/{fileName}` |
| 清单地址 | `https://raw.githubusercontent.com/market-of-labs/store/master/apps.json` | `https://<cf域>/manifest` |
| CF 隐藏红线 | ⚠️ **暂时失守**（真实 `owner/repo` 出现在清单里） | 满足 |

**客户端不需要任何改动**：它只消费清单里现成的 URL、从不自行拼前缀。切 CF = 改这一行 + 改伴侣应用设置里的清单地址，**不发新版应用**。

> `tagTemplate` / `assetNameTemplate` 是**契约常量**，两期都不变。改它们 = 改 Release 结构与文件命名契约。

---

## 6. 当前状态（**干净的起点**）

- `sources/` **是空的**（只有 `.gitkeep`），`apps.json` **不存在** —— 原先那份手写种子数据（3 条来源 + 一份清单）已整体清空，从头开始。
- 下一条真实数据来自**第一张新增 issue**：它落下 `sources/{appId}.json`、建 `{appId}` Release、把 APK 镜像进去、重建 `apps.json`。在那之前：
  - 设备端拉清单地址是 **404**（预期，不是故障）；
  - `forge-core` 的黄金测试 `TestGoldenRealRepo` 会**跳过**（它要求 `apps.json` 存在才跑）—— 码在、数据不在，跳过是对的行为。
- **⚠️ 两条会被"从头开始"带走的条目，要用时得重新收录**：
  - `com.obtainium.companion`（`kind:"companion"`，伴侣应用**自更新**的来源）—— 没有它，自更新链路没有上游；
  - `dev.imranr.obtainium`（`kind:"obtainium"`，**引导安装 Obtainium** 的候选）—— 没有它，伴侣应用找不到"该装哪一个"。
  这两条都不是自动回来的，各开一张新增单即可（前者上游指向本市场的 `companion` 仓库的 Release）。
- `com.github.HailLauncher` 也一并清掉了（它当初只是"待镜像"的占位）。

---

## 7. 部署前置检查

| # | 检查项 |
|---|---|
| **1** | **本仓库及其所属 org 的「Immutable releases」必须为 OFF**（2025-10-28 GA）。开启后 asset 不能增删改、tag 不能删/移 —— 本设计的追加上传与 `_incoming` 清场全部失效；删除不可变 Release 会**永久烧毁该 tag**，而 `tag = {appId}` 不可重建。**"先开后关"也不安全**（已固化的 Release 不受影响）。 |
| 2 | `store` 转私有后按私有仓库计费 —— 逻辑跑在公有的 `forge`（其实现来自私有的 `forge-core`），本仓库只跑一个 dispatch 步骤 |
| 3 | 本仓库与 `forge` 各自都能解析出 **同名** secret `GH_PAT`（**仓库级或组织级都行** —— 现状是本仓库用仓库级、`forge` 走组织级），值是**同一把** fine-grained PAT，仓库范围必须含 `store`(Contents RW + Issues RW) / `forge`(Contents RW) / **`forge-core`(Contents R)** |
| 3b | `forge-core` 必须**已发布过至少一个带 `forge-linux-amd64` asset 的 tag** —— 公有的 `forge` 浮动取 latest，一个 Release 都没有时取件直接失败 |
| 4 | 本仓库 workflow 保持 `permissions: {}` |
| 5 | 确认 `_incoming` 始终是 **draft** 且从未被误 Publish（CI 那条路每次上传前会先复位成 draft，见 §4；网页那条路靠人别点错） |
