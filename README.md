# store · 私有市场数据仓库

这是**私密应用市场的数据落地处**：输入（`sources/`）、产物（`store/`）、以及全部 APK 二进制（Releases）。

- **现在公有**（为了让开发期的 raw 直链可直接用）→ **部署期转私有**。
- 转私有正是 CF 遮蔽网关存在的理由，也是全部维护逻辑被拆到 [`forge`](../forge) 的原因（GitHub 对私有仓库的 Actions 分钟数计费）。
- 完整设计见 `market-spec/`（尤其 **03 · 维护逻辑**）。

> ⚠️ **这个仓库不执行任何维护逻辑**。它只跑一个「转手就发车」的极薄 workflow，把事件转告 `forge`。

---

## 1. 目录

```
store/
├── sources/                    # 输入 —— 唯一由人维护的事实源，一应用一文件
│   ├── dev.imranr.obtainium.json
│   └── com.obtainium.companion.json
├── apps.json                   # 最终清单 —— 放**根目录**，因为客户端唯一消费的就是它
├── store/                      # 其余产物 + 契约常量 —— 只有 forge 写
│   ├── index.json              # 版本/asset 机读索引（客户端不消费，D23）
│   └── endpoints.json          # 地址模板（第一期 ⇄ 部署期的唯一开关）
├── .github/
│   ├── workflows/forward.yml   # 薄转发：不 checkout、不插值、只 POST 一次 dispatch
│   └── ISSUE_TEMPLATE/         # 给小圈子成员用的两个入口
├── README.md
└── .gitignore
```

**`sources/` 是输入，产物是 `apps.json` 与 `store/` 两处。** 不要手工编辑 `apps.json` / `store/index.json`
—— 它们由 `forge` 生成，手改会在下一轮对账被覆盖。

> **为什么清单在根目录而不在 `store/` 里**：`apps.json` 是**唯一直接伺服给客户端**的文件，
> 它的路径就是设备的默认清单地址（`raw.githubusercontent.com/market-of-labs/store/master/apps.json`）。
> 把它放在根目录，地址里就不会出现 `store/store` 这种「仓库名与目录名重复」的段。
> `index.json` 客户端不消费（D23），`endpoints.json` 是给 forge 与 CF 读的契约常量，
> 两者都不需要短路径，留在 `store/` 里保持与输入面分开。
>
> ⚠️ **路径里的分支名是 `master`**（本仓库的默认分支），别顺手写成 `main`。
> `companion` 仓库用的是 `main`，两者**不同名是有意的** —— 对齐方式是改地址，不是改默认分支。
> 所以任何硬编码这条 raw 直链的地方（`companion/app/build.gradle.kts` 的内置默认值、
> 上述两处文档）都必须同时改，漏一处就是设备端一个 404。

---

## 2. 两条入口

### 甲 · 直接改文件 push（你自己用）

编辑/新增 `sources/{appId}.json` 后 push。`push` 到 `sources/**` 会触发 `forward.yml` → `forge` 收敛该条目。

新增一个应用 = 新建 `sources/{appId}.json`：

```json
{
  "id": "org.thirdparty.app",
  "name": "第三方公开示例",
  "author": "ThirdParty",
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

### 乙 · 开 issue（给小圈子成员用，不需要仓库写权限）

- **新增 · 标准源** → `.github/ISSUE_TEMPLATE/add-source.yml`
- **变更 · 移除** → `.github/ISSUE_TEMPLATE/change-source.yml`

`forge` 校验后**只改申请涉及的字段**、回评、关闭 issue。被拒绝也会写明原因。

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

> ⚠️ **为什么必须有「点 Publish」这一步**：往 Release 上传/改名/删 asset **不触发任何 `release` 事件**。Publish 是唯一能让流程发车的动作。
>
> ⚠️ **清场用「改回 draft」，绝不用「删 Release」**：删 Release 会让 tag 消失；若该仓库曾开启过 Immutable Releases，该 tag 会被**永久烧毁**且无法重建。

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

## 6. 当前状态（种子数据，待 `forge` 首次运行替换）

- `sources/` 已有 2 条：`dev.imranr.obtainium`（`kind:"obtainium"`，启动器安装候选）与 `com.obtainium.companion`（`kind:"companion"`，自更新来源）。
- `apps.json` / `store/index.json` 是**手写种子**，只为让伴侣应用在 `forge` 跑起来之前有东西可拉。`index.json` 里的 `size` 是占位 `0`。
- 一旦 `forge` 跑过一次，这两个产物会被真实数据覆盖。
- **现在所有 `apkUrls` 都指向尚不存在的 Release**（tag=`{appId}` 的 Release 还没建）—— 属预期，`forge` 首次收录时创建。

---

## 7. 部署前置检查

| # | 检查项 |
|---|---|
| **1** | **本仓库及其所属 org 的「Immutable releases」必须为 OFF**（2025-10-28 GA）。开启后 asset 不能增删改、tag 不能删/移 —— 本设计的追加上传与 `_incoming` 清场全部失效；删除不可变 Release 会**永久烧毁该 tag**，而 `tag = {appId}` 不可重建。**"先开后关"也不安全**（已固化的 Release 不受影响）。 |
| 2 | `store` 转私有后按私有仓库计费 —— 逻辑都在公有的 `forge` 跑，本仓库只跑一次 `curl` |
| 3 | `FORGE_DISPATCH_TOKEN`（本仓库 secret）与 `forge` 的 `STORE_WRITE_TOKEN` 是**同一把** fine-grained PAT |
| 4 | 本仓库 workflow 保持 `permissions: {}` |
| 5 | 确认 `_incoming` 始终是 **draft** 且从未被误 Publish |
