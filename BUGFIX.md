# 本版修复（BUGFIX）

## 标题

归档工具探测在 spawn 同步抛错时死于 TDZ：`Cannot access 'timer' before initialization`，
探测承诺以 ReferenceError 告终而不是「工具不可用」，且毒化进程级探测缓存。

## 现象与证据链

- 上游 issue [#995](https://github.com/omdsh-dev/dsh-better-sidebar/issues/995)
  报告用户侧出现裸错误 `Cannot access 'timer' before initialization`——这是
  JavaScript 的暂时性死区（TDZ）错误，只可能来自「在 `let`/`const` 初始化前读它」。
- 在 `src/archive-tools.ts` 的 `probeTool()` 中定位到全仓唯一吻合的模式：
  `finish()`（第 96 行）读取 `timer`，而 `const timer = setTimeout(...)` 在
  第 106 行才声明；当注入的 `spawner(...)` **同步抛错**时（`spawn` 本身在
  参数/选项非法时确实会同步抛，如 EINVAL），catch 块（第 103 行）调用
  `finish(false)`，此时 `timer` 尚在 TDZ 中。
- 已有测试用 EventEmitter 异步发 `'error'` 事件模拟「起不来」，从未覆盖
  同步抛错路径，所以该缺陷在上游测试全绿的情况下存活。
- 危害不止一次失败：`tryTools()` 把 `probeTool(...)` 的承诺缓存进进程级
  `availability` Map，被 reject 的承诺会被一直缓存——一次同步抛错后，
  本进程内所有归档（打包下载）请求都会重复抛同一个 ReferenceError。

### 修复前证据（最小复现，修复前运行）

向 `tryTools(['zip','tar'], spawner)` 注入一个同步抛错的 spawner，
期望 resolve 为 `undefined`（工具不可用、依次回退），实际：

```
AssertionError: promise rejected "ReferenceError: Cannot access 'timer' bef…"
  instead of resolving
Caused by: ReferenceError: Cannot access 'timer' before initialization
  ❯ finish src/archive-tools.ts:96:20
  ❯ src/archive-tools.ts:103:7
  ❯ probeTool src/archive-tools.ts:91:10
  ❯ tryTools src/archive-tools.ts:146:17
```

## 根因

`probeTool()` 把超时定时器声明在 `finish()` 闭包之后、且位于 try/catch 之后：
正常路径（child 异步发事件）里 `finish` 必然在定时器赋值后才被调用，
唯独 spawner 同步抛错这条路径会抢先调用 `finish`，读到 TDZ 中的 `timer`。

## 改动文件（+18 / −2，含测试）

1. `src/archive-tools.ts` —— 把 `timer` 改为在 `finish` 之前声明的
   `let timer: ReturnType<typeof setTimeout> | undefined`，spawn 成功后再赋值；
   `finish` 中 `if (timer !== undefined) clearTimeout(timer)`。
   其余逻辑（3000ms 探测超时、`settled` 单次结算、缓存策略）一字未动，
   并在声明处留注释说明为何必须提前声明。
2. `tests/archive-tools.spec.ts` —— 在 `tool probing (injected spawn)` 组内
   新增回归用例 `treats a spawner that throws synchronously as "tool absent"`，
   钉住该路径。

## 复验结果（修复后）

- 最小复现：由 reject ReferenceError 变为正常 resolve `undefined` ✅
- `tests/archive-tools.spec.ts`（含新增回归用例，23 个用例）、
  `tests/archive.spec.ts`（29）、`tests/archive-client.spec.ts`（4）全过 ✅
- `pnpm typecheck`（`tsc --noEmit`）零错误 ✅
- 全量 `pnpm test`：2680 通过 / 3 失败 / 17 跳过（共 2700）；
  3 个失败已核对为**基线既有失败**，非本修复引入（见文末附注）。

## 附注：基线失败核对

全量运行中失败的 3 个用例全部位于 git 子系统，与归档探测无关：

- `tests/git.spec.ts > git parsing > never lets a branch operand reach git as an option (checkout)`
- `tests/smoke.spec.ts > git destructive operations (scratch repository) > revert creates a revert commit`
- `tests/smoke.spec.ts > git destructive operations (scratch repository) > cherry-pick applies a commit from another branch`

核对方法：`git stash` 本版全部改动回到上游基线 `83faf70`，单独重跑
`tests/git.spec.ts` 与 `tests/smoke.spec.ts` —— 同样且仅这 3 个用例失败
（3 failed / 97 passed），与带修复运行的结果逐一对应。故为上游基线在
本机环境下的既有失败，本版不引入任何回归。
