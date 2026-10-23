# 本版修复（BUGFIX）· 3.2.1 → 3.2.2

## 标题

会话删除端点对 JSON `null` 请求体抛 TypeError：handler 直接 reject、一个字节的响应都不写，违背其自述的「永不带着打开的 socket reject」契约。

## 现象与根因

宿主半区端点 `POST /api/mobile-nav.session.delete`（`src/index.ts`）的处理顺序是：先在 try/catch 内 `JSON.parse(await readBody(req))`，出块后才解构取 id：

```ts
const { sessionId } = body
```

`null` 是合法 JSON：`JSON.parse('null')` 不抛错、返回 `null`，于是它顺利穿过解析守卫；随后对 `null` 解构抛 `TypeError: Cannot destructure property 'sessionId' of 'body' as it is null.`。这行在 try/catch **之外**，异常直接使 handler 返回的 Promise reject，响应从未写出——而该 handler 的源码注释明确写着这条契约：「this handler owns the response, so it may never reject with the socket open (the host's dispatcher would then answer a bare 400 with no body)」。其余畸形 body（数字、字符串、数组、缺字段对象）都会被原始值装箱/空解构兜住、落到结构化 400，唯独 `null`（以及 JSON 无法产生的 `undefined`）能把解构打穿。

## 证据链

1. **最小复现（离线，假宿主 ctx）**：脚本以桩 `HostContext` 调 `apply()` 捕获注册的 route handler，再以可读流喂入不同请求体直调 handler（脚本存于集成工作区 `.fixwork/web-mobile/repro.mjs`，未入库）。修复前（上游 lib 构建）输出：

   ```
   body=null -> rejection=TypeError: Cannot destructure property 'sessionId' of 'body' as it is null. status=undefined body=undefined
   body={"sessionId": 42} -> rejection=none status=400 body={"error":{"code":"invalid-session-id",...}}
   body={"sessionId": "x"} -> rejection=none status=404 body={"error":{"code":"session-not-found",...}}
   VERDICT=BUG (handler rejected)   # exit 1
   ```

   对照组证明路由、来源门与删除纯核都正常，唯独 `null` 一条把 handler 打 reject 且零响应。
2. **issue 面**：上游 open issue 共 2 条（#231 上下文圆圈与发送/停止按钮间距、#232 若干小问题与建议），均为界面建议、无一报告本缺陷；本 bug 由静态审查（解构点在守卫外）定位、复现坐实。
3. **基线测试**：`pnpm verify`（tsc 双配置）基线通过；`pnpm test:core` 基线 532 条中 531 过、1 红——`tests/delete-session.test.ts` 的 stash-stage 用例靠 `chmod 0500` 注入重命名失败，但沙箱以 root（uid 0）运行、权限位拦不住 rename，删除返回 200 而非用例预期的 500。以非 root 的 `nobody` 身份在 /tmp 副本单跑该文件为 **17/17 全过**，证明该红是 root 环境伪影、非产品缺陷，与本修复无关（本版未动 delete-session）。

## 改动

- `src/index.ts`（唯一实现改动，+5/−1）：解构改为 `const { sessionId } = body ?? {}`，并加注释说明 `null` 为何能穿过解析守卫。`null` body 自此落到既有的 id 校验、得到结构化 `400 invalid-session-id`，与其它无 id 的 body 形状口径一致。
- `lib/`：按仓库纪律由 `pnpm build` 重建（`lib/index.js` 及其 map 同步；client bundle 与基线逐字节一致、本版未动客户端）。
- `package.json`：version 3.2.1 → 3.2.2，repository.url 改指本 fork 落点仓；无其它版本戳需同步（lock 文件不含根包版本字段，README 中的 v3.2.1 是历史发版段落、属上游原文不动）。
- 新增 `UPSTREAM.md`、本文件；README 顶部追加 fork 头（原文一字未删）；LICENSE 原样保留。

## 复验结果

- 复现脚本修复后 exit 0：`body=null` → `rejection=none status=400 {"error":{"code":"invalid-session-id",...}}`，其余两条对照输出不变。
- `pnpm verify` 退出码 0；`pnpm test:core` 531/532，唯一红仍是上述 root 环境伪影（与基线同条、同因），零新增失败。
- `node scripts/client-bundle-smoke.mjs` PASS（61 模块内联、entry 正常加载）。

## 验证边界

- 未在仓库内新增该 handler 的回归测试文件：`src/index.ts` 的相对 import 使用编译产物式 `.js` 后缀，`node --test` 的 type-stripping 直跑无法解析（`ERR_MODULE_NOT_FOUND: src/compress.js`）——这也是上游 61 个测试无一 import 该文件的原因。回归证据由上述外部复现脚本承担（修复前 exit 1 / 修复后 exit 0，构造与输出已完整记录于此）。
- 未做真实 DSH 宿主 + 真机验收：本修复是请求入口的参数形状守卫，离线复现已覆盖其全部行为面（解析、解构、结构化响应）。
- `body` 的静态类型由 `as DeleteSessionBody` 断言给出，编译期看不出可空性；本版按运行时事实修守卫，未改该断言的类型声明（改类型会扩散到 handler 其余读法，超出最小改动）。
