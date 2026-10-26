# GH-GameGather UPSTREAM · 数据服务

> 📚 **本文档属于 GH 系列交接项目。** 总入口（全部 10 个仓库索引、可用性实测、可复制的 AI Agent prompt）：https://github.com/A13612812330/GH-Handover

> **`GH-GameGather-APP` 的配套上游服务。** 桌面端（Electron 壳）用它取实时数据；
> 不给它，App 就只能看离线快照 `demo/data.js`。
>
> 来源：从作者本地 `game-aggregator` 项目按 **`require` 闭包**裁剪后的可运行子集。
> 快照时点 **2026-10-26**。

---

## 一、它是什么

一个 Express 单进程服务，把两个源站（机地 `jidiyouxi.com` / XDGAME `xdgamer.com`）的
游戏信息聚合成 JSON 接口，外加本地 JSON 索引（MOD / 修改器 / 存档 / 设备规格字典）。

| 层 | 内容 |
|---|---|
| `server.js` 0.25 MB | 全部路由与业务编排 |
| `shared.js` | 与服务端共用的归一化逻辑（名字、日期、封面…） |
| `fetchers/` 15 个 | 两个源站的抓取器 + 截图去重 |
| `data/` 98 个文件 **93.6 MB** | 20+ JSON 索引快照（`mods.json` 23.4 MB / `spec-req.json` 12.9 MB / `jidi-topics.json` 10.3 MB / `games.json` 8.2 MB …）+ 设备规格字典 |
| `public/` 8 个文件 | 源站自己的网页（`index.html` / `roster` / `emulator` / `resources` / `extract` / `unpack` / `admin`） |

**运行时依赖只有 3 个**：`cheerio`、`express`、`js-yaml`。

---

## 二、跑起来

```bash
npm install
PORT=8199 node server.js          # Windows cmd: set PORT=8199 && node server.js
# → http://127.0.0.1:8199
```

不设 `PORT` 默认 **8123**；端口被占用时**不会自动换端口**（`GAMEHUB_PORT_SHIFT=1` 才允许 +1）。
默认只听 `127.0.0.1`（`GAMEHUB_HOST` 可改）。

自检：

```bash
curl http://127.0.0.1:8199/api/health
# {"ok":true,...,"pid":...,"startedAt":...,"host":"127.0.0.1","port":8199}

curl "http://127.0.0.1:8199/api/mods/stats"
# {"ok":true,"total":8943,"matched":8594,"matchedRate":96.1,...}
```

### 接到桌面端

壳（`electron/main.js`）用 `GAMEHUB_ROOT` 定位本服务，用 `GAMEHUB_PORT` 指定端口，
拉起子进程时把端口以 `PORT` 传给本服务：

```bash
export GAMEHUB_ROOT="/path/to/此目录"
GAMEHUB_PORT=8123 npm start        # 在 electron/ 下
```

---

## 三、能力边界（接手前必须知道）

| 接口 | 取数方式 | 没网会怎样 |
|---|---|---|
| `/api/mods/*`、`/api/trainers/*`、`/api/saves-yx/*`、`/api/device/*` | **纯读 `data/*.json`** | ✅ 正常，数据停在快照时点 |
| `/api/feed`、`/api/rank`、`/api/category`、`/api/detail` | **实时抓源站**（`fetchers/`） | ❌ 抓不到；且**源站一改版就会坏** |

所以：**这份代码复刻的是「采集与聚合能力」，不是「源站的实时内容」。**
慢启动是正常的 —— `feed` / `rank` 冷启动 20 s 以上，热后 1~2 ms。

---

## 四、凭据（本仓库刻意不含）

管理后台的账号与令牌**没有入库**，`server.js` 两条读取路径都是 `try/catch`，缺文件不会崩，
只是管理页一律 401。要用就设环境变量：

```bash
GAMEHUB_ADMIN_USER=yourname
GAMEHUB_ADMIN_PASS=yourpassword
GAMEHUB_ADMIN_TOKEN=any-random-string   # 给脚本/无头测试用
```

`data/magnet-config.json` 里的 `secret` 已换成随机占位值，接本机磁力引擎时自行改。

---

## 五、边界声明

- 只做**信息聚合与索引**：不托管、不分发任何游戏文件；条目一律指向原始出处。
- 抓取对象是两个第三方站点，**对方改版、限流或封 IP 都会导致对应接口失效**，
  这是这类聚合服务的固有属性，不是 bug。
- 请自行确认在你的使用场景下抓取这些站点是否符合当地法律与对方服务条款。

---

## 六、目录之外的裁剪说明

作者本地原项目（1.4 GB / 17,779 文件）里还有 `tools/`（采集脚本）、`docs/`、
`_ref` / `_bak` / `_test-out` / `_probe-fg-dl` 等开发期目录，**运行期一律不需要**，故未包含。

数据侧已剔除：`data/_*`（原始抓取中间产物 42.8 MB）、`*.bak-*`、审计日志、aria2 会话文件；
`public/dl/` 下 92.7 MB 的第三方安装包也已剔除。
