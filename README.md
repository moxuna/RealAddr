# RealAddr — 真实地址生成器（Cloudflare Worker）

线上地址：https://address.heartwopen.workers.dev/

随机生成**真实存在**的门牌级地址（数据来自 OpenStreetMap），并配套一条与地址所在城市严格对应的电话号码。地址数据已打包进 Worker，冷启动也能秒回，不依赖任何第三方接口，页面不加载境外资源。

## 功能

- **地址绝对真实**：门牌级数据取自 OSM，每条都带 OSM 元素 id，可在页面底部逐条核对
- **电话与地址对应**：先选城市，再按该城市固话区号（罗马 06、莫斯科 495）或合法手机号段生成
- **每次随机换城市**：同一国家连续抽取会命中不同城市，不会总停在同一条街
- **省/州筛选**：24 个国家/地区全部可按下一级行政区抽取，切换国家时下拉自动换成对应的省/州/大区，默认「全部」；下拉按名称排序（美国按州代码字母序，其余按中文拼音），便于查找
- **免税地区标注**：不征收消费税的次级行政区在名称后标「（免税）」，如美国 OR、DE、NH、MT、AK 与香港全境；凭说明可自行核对
- **免税州能力**：美国可只生成不征收州销售税的州（OR、DE、NH、MT、AK），页面不提供开关，直接用接口 `taxFree=1` 调用
- **类型筛选**：住宅 / 商业 / 不限
- 上游全部不可用时，仍可从内置数据或存档返回；内置数据覆盖 24 个国家/地区

## 接口

```
GET /api?country=US&type=residential|commercial|any[&taxFree=1][&state=OR]
GET /api/states?country=US
```

- `state` 支持省/州/大区代码或名称（如 `OR`、`Oregon`、`俄勒冈州`、`广东省`），大小写不敏感；名称不存在时返回 400
- `taxFree=1` 仅对美国有效，与 `state` 互斥（指定州时该州必须免税）；页面已移除免税州开关，此参数只在直接调用接口时使用
- `/api/states` 返回 `{ regionLabel, states: [{ code, name, count, taxFree?, note? }] }`，页面下拉用它渲染；`taxFree` 为真时页面标「（免税）」

## 数据规模（2026-10-15，共 94,785 条）

| 国家 | 条数 | 国家 | 条数 |
|---|---|---|---|
| US 美国 | 10,000 | CN 中国 | 10,000 |
| TR 土耳其 | 10,000 | DE 德国 | 6,000 |
| IT 意大利 | 4,894 | UK 英国 | 4,841 |
| TW 台湾 | 4,800 | CA 加拿大 | 4,800 |
| FR 法国 | 4,232 | ES 西班牙 | 4,011 |
| AU 澳大利亚 | 3,774 | NG 尼日利亚 | 3,598 |
| JP 日本 | 2,855 | RU 俄罗斯 | 2,578 |
| HK 香港 | 2,433 | MX 墨西哥 | 2,354 |
| AR 阿根廷 | 2,400 | EG 埃及 | 2,400 |
| SA 沙特 | 2,346 | BR 巴西 | 1,893 |
| KR 韩国 | 1,365 | IN 印度 | 1,168 |
| ID 印尼 | 1,035 | ZA 南非 | 1,008 |

每国 1,000 至 10,000 条，全部来自 OSM 真实门牌。

## 仓库文件

| 文件 | 说明 |
|---|---|
| `worker.js` | **部署只用这一个文件**，内置 94,785 条真实地址 |
| `wrangler.toml` | Worker 名 `address` 的配置，面板 Git 部署与命令行部署都要用 |
| `README.md` / `优化说明.md` | 文档 |

数据抓取与注入脚本（harvest.cjs、inject-bundle.cjs 等）和 12 MB 的 bundle_data.json 只保留在本地开发目录，不随仓库发布。

## 部署（两种方式，都不需要命令行）

### 方式一：面板 Git 自动部署（推荐，设置一次长期有效）

1. 登录 Cloudflare 面板，进入 Workers 和 Pages，点开 `address`
2. Settings（设置）→ Builds（构建）→ Connect（连接），授权 GitHub 账号 moxuna
3. 选择仓库 `moxuna/RealAddr`，生产分支选 `main`
4. 构建命令留空，部署命令填 `npx wrangler deploy`，保存
5. 之后仓库一有新提交，Cloudflare 会自动重新部署，线上地址不变

免费版即可使用，每月 3,000 构建分钟。

### 方式二：面板手动粘贴

Workers 和 Pages → `address` → 编辑代码 → 粘贴 `worker.js` 全文 → 部署。
worker.js 约 10 MB，网页编辑器粘贴大文件容易卡顿甚至失败，建议只在应急时使用。

### 命令行方式（可选）

```bash
NODE_USE_ENV_PROXY=1 HTTPS_PROXY=http://127.0.0.1:10808 wrangler deploy
```

## 数据来源

地址数据来自 [OpenStreetMap](https://www.openstreetmap.org/) 贡献者，遵循 ODbL 许可。电话号码由程序随机生成，与真实住户无关。
