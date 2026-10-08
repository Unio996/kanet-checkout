# KANet 结账页（checkout-static）发布包

纯静态、无服务器依赖的浏览器结账页：买家打开一个带签名报价的链接，页面在浏览器本地用真实
kaspa-wasm + silverc-wasm 验签、推导订单地址、生成收款地址+扫码二维码，到账后可直连节点监视状态并
触发分账/退款——不需要任何后端进程。

**本包所有仓库内文件均取自 commit `fe37797ec0cb8b7bbc91216ba1114309b1f0075f`，用 `git show <commit>:<path>` 原样提取，与该 commit
逐字节一致——不是手工编辑的临时副本。**

## 这个包里有什么

```
checkout-static/
├── checkout.html              买家打开的页面
├── checkout.js                页面胶水层（解析链接/展示进度/驱动收款+监视+触发流程）
├── verify-core.js             验签/验证链逻辑
├── resolve-order-browser.js   订单地址推导（角色解析）
├── resolve-order-wasm.js      订单地址推导主路径（真 silverc 编译器）
├── order-template.js          订单地址推导备选路径（固定偏移覆写，silverc-wasm 加载失败时自动降级）
├── monitor.js                 订单地址到账状态/确认深度/节点 PMT 只读监视（浏览器直连节点 wss）
├── broadcast-commission.js    触发分账/退款交易组装（零签名，covenant 脚本本身是判据）
└── vendor/
    ├── kaspa-web/              浏览器版 kaspa-wasm（sha256 见下）
    ├── silverc-wasm/           浏览器版 silverc 编译器（sha256 见下）
    ├── noble-hashes/           vendored blake2b（浏览器原生 ESM 需要，MIT 协议，见目录内 LICENSE）
    ├── qrcode-generator/       扫码付款二维码编码器（MIT，Kazuhiko Arase，未改动上游代码）
    ├── generic-entry-witness-browser.mjs  covenant 签名 witness ABI 编码器（触发分账/退款用）
    ├── tx-mass-ub-browser.mjs  广播前三维 mass 预检
    ├── sil-source/             .sil 合约源码（订单地址推导需要读源码文本）
    └── fee-split-browser.mjs   分成计算逻辑（与仓库内 fee-split.mjs 逐字节同步）
```

**不在这个包里**（如实说明为什么）：
- `config.html`/`config.js`（商家生成签名报价链接的工具）和 `resolver.mjs`（商家侧签名服务）——
  这两者是**商家侧**工具，仍需要一个本机 Node 进程（`resolver.mjs` 监听 `127.0.0.1`，只在本机内网
  可达，私钥仅内存中用一次不落盘）来对商家私钥签名，跟这个包"买家侧纯静态、零后端"的定位不同。
  商家如需生成报价链接，用仓库内 `kasia-console/src/lib/checkout-static/config.html` + 起
  `resolver.mjs` 的既有流程。（⑩：商家侧改成浏览器内 kaspa-wasm 本地签名后会并入下一版发布包。）
- `wasm-pin-check.mjs`、`fee-split-browser-parity.mjs`、
  `vendor/generic-entry-witness-browser-parity.mjs`、`vendor/tx-mass-ub-browser-parity.mjs`——
  仓库内部的开发期自检脚本，买家打开页面时用不到（离开仓库目录结构就是死代码：裸 import
  `kaspa-wasm` + 相对路径指回仓库内其他模块）。

## 编译器加载失败时怎么办

silverc-wasm 加载失败（网络/浏览器兼容性问题）时，页面会自动降级到固定偏移覆写路径继续完成订单
地址推导（该路径独立验证过），并展示清楚的失败原因 + 三条替代方式（换浏览器重试 / 自行托管这份
发布包 / 自行运行 `resolver.mjs`）——降级路径下触发分账/退款功能不可用（需要真编译器才有
covenant entry ABI）。

## 产物指纹（部署前校验，`checkout.js` 运行时也会自己核对一次 sha256）

| 文件 | sha256 |
|---|---|
| `vendor/kaspa-web/kaspa_bg.wasm` | `732bdaa3ee8353c026654e9c7dd729674eb1bd064e8a0b8927b4cfb7df859e51` |
| `vendor/silverc-wasm/silverc_lang_bg.wasm` | `868e3f1b247a02b2a2eb39350dfcdd2fe03157c57d99b803b7e9ab2d7bfdc325` |
| `checkout-static/vendor/kaspa-web/kaspa.js` | `d0f14833f2db9c2fb4145a40e68d6221862a667eef15512e1994c02a5e588b04` |
| `checkout-static/vendor/kaspa-web/kaspa.d.ts` | `78628cf0abe112d4103e3c66499e49e12881be1cd24f0953076de55f021e1b5d` |
| `checkout-static/vendor/kaspa-web/LICENSE` | `929edb70a6ba89ae05689d3ee33453fdd63717a9f592038bf3d9420e35b434e6` |
| `checkout-static/vendor/silverc-wasm/silverc_lang.js` | `d67a0977cb97e74f9ed4f2808b29bb1182397aec7267c967528d6af0a535aaf1` |
| `checkout-static/vendor/silverc-wasm/silverc_lang.d.ts` | `f03750534beb4916cef57a070f3f77aee11786eb87042d8ae8fe2431bedfef05` |
| `checkout-static/vendor/silverc-wasm/silverc_lang_bg.wasm.d.ts` | `9cc7c87aff66a4a68bda5f1847bc532d75efc6ce4f10f096a63da318af7c53c1` |

来源、构建命令、goldenSample 同源判据见各自 `vendor/*/README.md`。以上全部指纹(含绑定文件)锚定在
仓库内 `scripts/kaspa-wasm-web-pin.json`/`scripts/silverc-wasm-pin.json`，本发布包由打包脚本
fail-closed 逐个核对过，不匹配不会生成包。

## 部署要求（真实实测，非估算）

1. **`Content-Type: application/wasm`** — 现用代码路径（`fetch()` + `WebAssembly.compile(bytes)`，
   非 `instantiateStreaming`）对这个头不敏感，但仍建议显式配置对，为将来切换到流式编译留余地。
2. **启用 br（Brotli）压缩，视为硬性部署要求，不是可选优化** — 两个 wasm 文件未压缩合计约 16.7MB，
   br 压缩后约 5.07MB。真实 Slow-3G(400kbps)+4x CPU 限速下实测两种流程总耗时：**br 关 347.2 秒
   （5.8 分钟）vs br 开 117.7 秒（2.0 分钟），加速比 2.9x**。不开 br 不是页面代码 bug，是这两份
   wasm 产物大小决定的物理下限，但会让弱网用户等待接近 6 分钟。
3. **长效不可变缓存**（`Cache-Control: public, max-age=31536000, immutable`）— 两个 wasm 文件用内容
   sha256 锚定，文件不变则可以放心长期缓存；升级版本必须同时改 pin 常量 + 发新文件（不能只换文件
   不改 pin，会被运行时校验拒绝，这是设计如此，不是需要绕过的限制）。
4. **CSP（若宿主环境设置了严格 Content-Security-Policy）**：`script-src` 需要带
   `'wasm-unsafe-eval'`（Chrome 90+/Firefox 79+）以允许 `WebAssembly.compile`/`instantiate`。
5. **必须走 `http(s)://` 托管**，不支持 `file://` 直接双击打开。
6. **下载进度反馈已内置**（本发布包版本）：`checkout.js` 会实时显示"下载中… X.X / Y.Y MB (NN%)"，
   压缩传输下（Content-Length 反映压缩前字节、与解压后收到字节不可比）自动降级为"已收到 X.X MB"
   文案，不会出现进度超 100% 的情况。

## 浏览器直连公共节点（可选，用于查询链上数据；广播交易仍走 `submitTransaction` 走 RPC）

真实测试确认可用、无 CORS 问题、按直连延迟排序：

| 端点 | 直连耗时 | `getServerInfo` 耗时 | 版本 |
|---|---|---|---|
| `wss://sara.kaspa.red/kaspa/mainnet/wrpc/borsh` | 405ms | 189ms | 2.1.0 |
| `wss://nina.kaspa.blue/kaspa/mainnet/wrpc/borsh` | 400ms | 185ms | 2.1.0 |
| `wss://eva.kaspa.green/kaspa/mainnet/wrpc/borsh` | 459ms | 218ms | 2.0.1 |
| `wss://vivi.kaspa.blue/kaspa/mainnet/wrpc/borsh` | 532ms | 249ms | 2.1.0 |
| `wss://isla.kaspa.red/kaspa/mainnet/wrpc/borsh` | 609ms | 294ms | 2.1.0 |

确认不可用（CORS 拦截）：`noah.kaspa.blue`、`sean.kaspa.stream`、`adam.kaspa.green`。

**⚠️ 已知信号，部署前请知悉**：上表 5 个端点的域名命名模式（`<人名>.kaspa.<red/blue/green/stream>`）
提示它们可能来自同一个或一小撮协同的运营方，而不是 5 个互相独立的信任源——如果这个运营方的基础
设施出问题，5 个候选可能同时失效。**建议**：① 把这份清单当加速用的默认候选池，不要当"5 个独立
故障域"意义上的冗余；② 保留 `Resolver` 自动发现作为兜底（这份清单里的端点连不上时才用），不要
因为直连快就完全弃用自动发现；③ 这份清单需要定期人工复核，公开节点的可用性/版本/CORS 配置会变
（`eva.kaspa.green` 当前版本 2.0.1 比其他几个的 2.1.0 略旧）。

## 如实标注的覆盖缺口

- 真 Safari(macOS/iOS)、真 Android/iOS 设备、Telegram/WhatsApp/微信内置浏览器：本轮开发环境是
  Windows 桌面机，没有这些设备/App，未测试，需要有对应设备的人补测。
- 严格 CSP 环境下的实际行为：未起过带 CSP header 的测试服务器验证。
- ⑧d 的耗时数字是单次真实运行，非多轮统计中位数。
- ⑨ 只采样验证了 5 个可用端点，公开节点池可能还有更多未穷举。
