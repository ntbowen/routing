已保存到 Memo ✅

---

# cjdns v22.3 编译修复完整记录

> 环境：zagwrt · x86_64 · musl · feeds/routing/cjdns · 2026-04-27

---

## 问题现象

`cjdroute` 被错误链接至宿主机 glibc，目标设备无法运行：

```bash
# ❌ 错误
interpreter /lib64/ld-linux-x86-64.so.2   ← 宿主机 glibc

# ✅ 正确
interpreter /lib/ld-musl-x86_64.so.1      ← musl
```

---

## 根因分析

| 根因 | 说明 |
|------|------|
| **`do` 脚本未传 `--target`** | `cargo build` 无 `--target`，默认编译宿主机 native 目标，`CARGO_TARGET_*_LINKER` 环境变量完全失效 |
| **产物路径硬编码** | `do` 脚本 `move` 硬编码 `./target/$path/`，加 `--target` 后产物实际在 `./target/<triple>/$path/` |

---

## 修复方案

### 修复一：`003-do-cross-target.patch`

读取 `CARGO_TARGET_TRIPLE` 环境变量，动态添加 `--target` 并自适配产物路径：

```patch
--- a/do
+++ b/do
@@ -9,12 +9,18 @@ fi
 release="--release"
 path="release"
+target_arg=""
+target_dir=""
 if echo "$@" | grep -q '\-\-debug'; then
     release=""
     path="debug"
 fi 
-RUSTFLAGS="$RUSTFLAGS -g" $CARGO build $release
-if [ "$NO_TEST" = '' ]; then
-  RUST_BACKTRACE=1 "./target/$path/testcjdroute" all >/dev/null
+if [ -n "${CARGO_TARGET_TRIPLE:-}" ]; then
+    target_arg="--target $CARGO_TARGET_TRIPLE"
+    target_dir="$CARGO_TARGET_TRIPLE/"
 fi
+RUSTFLAGS="$RUSTFLAGS -g" $CARGO build $release $target_arg
+if [ "$NO_TEST" = '' ] && [ -z "${CARGO_TARGET_TRIPLE:-}" ]; then
+  RUST_BACKTRACE=1 "./target/$path/testcjdroute" all >/dev/null
+fi
 
-move "./target/$path/cjdroute" ./cjdroute
-move "./target/$path/cjdnstool" ./cjdnstool
+move "./target/${target_dir}$path/cjdroute" ./cjdroute
+move "./target/${target_dir}$path/cjdnstool" ./cjdnstool
```

### 修复二：Makefile 注入 `CARGO_TARGET_TRIPLE`

```makefile
	CARGO_TARGET_TRIPLE="x86_64-unknown-linux-musl" \        ← 新增
	CARGO_TARGET_X86_64_UNKNOWN_LINUX_MUSL_LINKER="$(TARGET_CC_NOCACHE)" \
	CARGO_TARGET_X86_64_UNKNOWN_LINUX_MUSL_AR="$(TARGET_AR)" \
```

---

## 补丁列表（最终）

| 文件 | 作用 |
|------|------|
| `001-five-mins-builder-zonk.patch` | 修正 `node_build/builder.js` 超时 |
| `002-make-js-sodium-include-dir.patch` | 注入 libsodium 头文件路径 |
| `003-do-cross-target.patch` | 使 `do` 脚本支持 `--target` 及路径自适配 ✅ 新增 |

---

## 验证结果

编译成功 ✅

```bash
file build_dir/target-x86_64_musl/cjdns-cjdns-v22.3/cjdroute
# 期望：interpreter /lib/ld-musl-x86_64.so.1
```

---

## 经验总结

> **Cargo 交叉编译铁律**：必须显式传 `--target`，否则 `CARGO_TARGET_<TRIPLE>_LINKER` 等环境变量完全无效。加了 `--target` 后产物路径从 `target/<profile>/` 变为 `target/<triple>/<profile>/`，构建脚本的路径逻辑必须同步调整。
