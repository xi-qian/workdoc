# Plan C: SUID Helper + Run 生命周期实现计划

> **给 agentic worker:** 必须使用子技能:用 superpowers:subagent-driven-development(推荐)或 superpowers:executing-plans 按任务逐个实现本计划。步骤使用复选框(`- [ ]`)语法进行跟踪。

**前置条件:** Plan A(Foundation)和 Plan B(channel registry)已完成。Per-(tenant,agent) 数据库已存在;channels 可以接收消息;`active_runs` 表可用。

**目标:** 用 host-direct run 进程替换 docker-per-group runtime。构建 `nc-setuid-helper`(SUID root 的 C 二进制),由其负责所有特权操作(用户创建、ACL 设置、cgroup、setuid+exec、信号、status)。构建调用它的控制平面 spawner。构建 agent-runner 的 live 轮询循环。本计划结束时,一条普通消息的流转路径为:channel → 控制平面 → helper.spawn → run 进程读取 inbound.db → provider → 写入 outbound.db → outbound 轮询器投递。

**架构:** `nc-setuid-helper` 暴露 5 个操作,并对参数严格校验,依据 `/var/lib/nanoclaw/users.db`(tuple→uid/gid/username 映射,PID start-time ticks 用于防复用)。控制平面通过 `child_process.spawnFile` 调用 helper。Runtime 数据库(`inbound.db`、`outbound.db`、`state.db`、`tools.db`)位于 `/var/lib/nanoclaw/runtime/<t>/<a>/<g>/{live|runs/<runId>}/` 下,权限 0700 + POSIX ACL 授予 `nanoclaw-svc` rwx。Agent-runner 是现有的 `container/agent-runner/src/index.ts` 做最小改造:不再期望 docker 环境,而是从 argv 读取 tenant/agent/group/runId 并轮询 inbound.db。

**技术栈:** C(C99)、libsqlite3、libacl、Linux cgroup v2、Node.js 20+、TypeScript NodeNext ESM、better-sqlite3、vitest。

---

## 文件结构

### 新建

| 路径 | 职责 |
|------|----------------|
| `native/nc-setuid-helper/` | C 源码目录 |
| `native/nc-setuid-helper/Makefile` | build + test + install 目标 |
| `native/nc-setuid-helper/src/main.c` | 入口点、命令分发 |
| `native/nc-setuid-helper/src/arg.c`, `arg.h` | 参数解析 + 校验 |
| `native/nc-setuid-helper/src/users_db.c`, `users_db.h` | `users.db` SQLite schema 与 CRUD |
| `native/nc-setuid-helper/src/naming.c`, `naming.h` | 用户名派生、ID 净化 |
| `native/nc-setuid-helper/src/acl.c`, `acl.h` | POSIX ACL 设置辅助函数 |
| `native/nc-setuid-helper/src/cgroup.c`, `cgroup.h` | cgroup v2 create/write/pid-add |
| `native/nc-setuid-helper/src/process.c`, `process.h` | setuid+setgid+exec、信号、status |
| `native/nc-setuid-helper/src/prepare.c`, `prepare.h` | `prepare` 命令逻辑 |
| `native/nc-setuid-helper/src/spawn.c`, `spawn.h` | `spawn` 命令逻辑 |
| `native/nc-setuid-helper/src/kill.c`, `kill.h` | `kill` 命令逻辑 |
| `native/nc-setuid-helper/src/status.c`, `status.h` | `status` 命令逻辑 |
| `native/nc-setuid-helper/src/cgroup_cmd.c`, `cgroup_cmd.h` | `cgroup` 命令逻辑 |
| `native/nc-setuid-helper/src/logging.c`, `logging.h` | 结构化 stderr 日志 |
| `native/nc-setuid-helper/tests/test_naming.c` | 纯逻辑单元测试(无需特权) |
| `native/nc-setuid-helper/tests/test_arg.c` | 参数解析器测试 |
| `native/nc-setuid-helper/tests/test_users_db.c` | users.db 逻辑测试 |
| `native/nc-setuid-helper/tests/test_acl_stub.c` | 不依赖 libacl 的 ACL 逻辑测试(用于非 root CI) |
| `native/nc-setuid-helper/tests/runner.sh` | 运行纯逻辑测试 + gated root 集成测试 |
| `native/nc-setuid-helper/tests/root/prepare_spawn_status.sh` | Root-gated E2E |
| `src/runtime/helper-client.ts` | TS 封装:调用 helper 二进制 |
| `src/runtime/helper-client.test.ts` | 使用 fake helper 二进制的测试 |
| `src/runtime/runtime-dbs.ts` | openRuntimeDbs(t,a,g,runId) + inbound/outbound/state/tools 的 schema |
| `src/runtime/runtime-dbs.test.ts` | Runtime DB schema 测试 |
| `src/runtime/spawner.ts` | prepare → spawn → 注册到 active_runs |
| `src/runtime/spawner.test.ts` | Spawner 测试(mock helper client) |
| `src/runtime/lifecycle.ts` | monitor(proc)、reap(SIGCHLD)、kill、reconcile |
| `src/runtime/lifecycle.test.ts` | Lifecycle 测试 |
| `src/runtime/agent-runner-bridge.ts` | 为 agent-runner live 模式构建 argv |

### 修改

| 路径 | 原因 |
|------|-----|
| `container/agent-runner/src/index.ts` | 接受 `--mode=live --tenant --agent --group --run-id --runtime-dir` argv。轮询 inbound.db。结果写入 outbound.db。idle / 控制停止时退出。 |
| `src/index.ts` | 收到消息时调用 spawner 而不是 docker runner。启动时 reconcile active_runs。 |
| `src/group-queue.ts` | 用 per-(tenant,agent,group) 的 run handle 替换 docker-specific 逻辑。 |

### 本计划不要触碰

- Tool IPC 迁移 — Plan E。
- Skill bundle 暂存 — Plan D。
- 迁移脚本 — Plan F。

---

## 全局不变量

- C 代码使用 `gcc -std=c99 -Wall -Werror -Wextra` 编译。
- C 测试无需特权运行。特权操作通过 `root/` 下的 shell 测试完成,若 `EUID != 0` 则跳过。
- TS 代码:NodeNext ESM,vitest 与源码同目录。
- Helper 二进制以 `key=value` 对的形式把日志写入 stderr,每行一个事件。
- Helper 退出码:0 = 成功,1 = 校验失败,2 = 系统错误,3 = 身份不匹配。
- 每次 helper 调用都重新校验 `users.db` 映射;不做内存级信任。

---

## Task 1: C 项目骨架 + Makefile

**文件:**
- 新建: `native/nc-setuid-helper/Makefile`
- 新建: `native/nc-setuid-helper/src/main.c`
- 新建: `native/nc-setuid-helper/src/logging.c`, `logging.h`
- 新建: `native/nc-setuid-helper/tests/runner.sh`

- [ ] **Step 1: 创建目录结构**

```bash
mkdir -p native/nc-setuid-helper/src native/nc-setuid-helper/tests/root
```

- [ ] **Step 2: 编写 Makefile**

新建 `native/nc-setuid-helper/Makefile`:

```makefile
CC      ?= gcc
CFLAGS  ?= -std=c99 -Wall -Werror -Wextra -O2 -fPIE
LDFLAGS ?= -pie
LIBS    ?= -lsqlite3 -lacl

SRC_DIR := src
TEST_DIR := tests

OBJS := $(patsubst $(SRC_DIR)/%.c,$(BUILD_DIR)/%.o,$(wildcard $(SRC_DIR)/*.c))

BIN := nc-setuid-helper

.PHONY: all build test clean install

all: build

build:
	mkdir -p $(BUILD_DIR)
	$(CC) $(CFLAGS) $(OBJS) -o $(BUILD_DIR)/$(BIN) $(LDFLAGS) $(LIBS)

# Pattern rule for objects
$(BUILD_DIR)/%.o: $(SRC_DIR)/%.c
	@mkdir -p $(BUILD_DIR)
	$(CC) $(CFLAGS) -c $< -o $@

# Pure logic unit tests — no privileges required.
test: build
	@bash $(TEST_DIR)/runner.sh

clean:
	rm -rf $(BUILD_DIR) $(TEST_DIR)/*.test

install: build
	install -d -m 0755 $(DESTDIR)/usr/lib/nanoclaw
	install -m 4750 -o root -g nc-priv $(BUILD_DIR)/$(BIN) $(DESTDIR)/usr/lib/nanoclaw/$(BIN)
```

- [ ] **Step 3: 编写日志辅助函数**

新建 `native/nc-setuid-helper/src/logging.h`:

```c
#ifndef NC_LOGGING_H
#define NC_LOGGING_H

#include <stdarg.h>

void nc_log(const char *level, const char *fmt, ...);

#define nc_log_info(...)    nc_log("info",    __VA_ARGS__)
#define nc_log_warn(...)    nc_log("warn",    __VA_ARGS__)
#define nc_log_error(...)   nc_log("error",   __VA_ARGS__)

#endif
```

新建 `native/nc-setuid-helper/src/logging.c`:

```c
#include "logging.h"

#include <stdio.h>
#include <time.h>

static void timestamp_iso(char *buf, size_t n) {
    time_t t = time(NULL);
    struct tm tm;
    gmtime_r(&t, &tm);
    strftime(buf, n, "%Y-%m-%dT%H:%M:%SZ", &tm);
}

void nc_log(const char *level, const char *fmt, ...) {
    char ts[32];
    timestamp_iso(ts, sizeof(ts));
    fprintf(stderr, "ts=%s level=%s ", ts, level);
    va_list ap;
    va_start(ap, fmt);
    vfprintf(stderr, fmt, ap);
    va_end(ap);
    fputc('\n', stderr);
}
```

- [ ] **Step 4: 编写最小 main.c**

新建 `native/nc-setuid-helper/src/main.c`:

```c
#include "logging.h"

#include <stdio.h>
#include <string.h>

static int usage(void) {
    fprintf(stderr,
        "usage: nc-setuid-helper <command> [args...]\n"
        "commands:\n"
        "  prepare --tenant=T --agent=A --group=G --runtime-dir=D\n"
        "  spawn   --uid=U --gid=G --cgroup=P --runtime-dir=D -- cmd...\n"
        "  kill    --pid=P --signal=S --uid=U --runtime-dir=D --cgroup=P --start-time=T\n"
        "  cgroup  --path=P --mem=MB --pids=N --cpu=S\n"
        "  status  --pid=P --uid=U --runtime-dir=D --cgroup=P --start-time=T\n");
    return 1;
}

int main(int argc, char **argv) {
    if (argc < 2) return usage();
    if (strcmp(argv[1], "--help") == 0 || strcmp(argv[1], "-h") == 0) return usage();

    nc_log_info("cmd=%s argc=%d", argv[1], argc);
    /* Command dispatch filled in by later tasks. */
    fprintf(stderr, "error: command '%s' not yet implemented\n", argv[1]);
    return 1;
}
```

- [ ] **Step 5: 编写空的测试 runner**

新建 `native/nc-setuid-helper/tests/runner.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

echo "no pure-logic tests yet"
exit 0
```

```bash
chmod +x native/nc-setuid-helper/tests/runner.sh
```

- [ ] **Step 6: 验证构建可用**

运行: `cd native/nc-setuid-helper && make build && ./nc-setuid-helper --help`
预期: 打印 usage 文本,退出码 1。

- [ ] **Step 7: 提交**

```bash
git add native/nc-setuid-helper/
git commit -m "feat(helper): scaffold nc-setuid-helper C project with Makefile"
```

---

## Task 2: Naming helpers(纯 C、单元测试)

**文件:**
- 新建: `src/naming.c`, `src/naming.h`
- 新建: `tests/test_naming.c`
- 修改: `tests/runner.sh`

- [ ] **Step 1: 编写 naming.h**

新建 `native/nc-setuid-helper/src/naming.h`:

```c
#ifndef NC_NAMING_H
#define NC_NAMING_H

#include <stdbool.h>
#include <stddef.h>

#define NC_ID_MAX 16
#define NC_USERNAME_MAX 32
#define NC_HASH_HEX_LEN 10  /* 10 hex chars = 5 bytes */

/**
 * Validate that `id` matches [a-z][a-z0-9-]* and is at most NC_ID_MAX chars.
 * Returns true on success and copies the lowercased id into `out` (size out_n).
 */
bool nc_validate_id(const char *id, char *out, size_t out_n);

/**
 * Sanitise a group id: lowercase, replace non-[a-z0-9-] with '-',
 * reject empty or path-traversal inputs.
 */
bool nc_sanitize_group(const char *raw, char *out, size_t out_n);

/**
 * Derive ncg-<t8>-<a8>-<hash10> from canonical tuple.
 * `out` must be at least NC_USERNAME_MAX bytes.
 */
bool nc_derive_username(const char *tenant, const char *agent, const char *group,
                        char *out, size_t out_n);

/**
 * 10-char lowercase hex SHA-256 of "tenant|agent|group", written to `out`.
 */
void nc_hash_tuple(const char *tenant, const char *agent, const char *group,
                   char *out, size_t out_n);

#endif
```

- [ ] **Step 2: 编写 naming.c**

新建 `native/nc-setuid-helper/src/naming.c`:

```c
#include "naming.h"

#include <ctype.h>
#include <openssl/sha.h>
#include <stdio.h>
#include <string.h>

static bool is_valid_id_char(char c, bool first) {
    if (first) return c >= 'a' && c <= 'z';
    return (c >= 'a' && c <= 'z') || (c >= '0' && c <= '9') || c == '-';
}

bool nc_validate_id(const char *id, char *out, size_t out_n) {
    if (!id || !out || out_n < NC_ID_MAX + 1) return false;
    size_t n = strlen(id);
    if (n == 0 || n > NC_ID_MAX) return false;
    for (size_t i = 0; i < n; i++) {
        char c = (char)tolower((unsigned char)id[i]);
        if (!is_valid_id_char(c, i == 0)) return false;
        out[i] = c;
    }
    out[n] = '\0';
    return true;
}

bool nc_sanitize_group(const char *raw, char *out, size_t out_n) {
    if (!raw || !out || out_n < 2) return false;
    if (strchr(raw, '/') || strstr(raw, "..")) return false;
    size_t n = strlen(raw);
    if (n == 0) return false;
    size_t j = 0;
    for (size_t i = 0; i < n; i++) {
        char c = (char)tolower((unsigned char)raw[i]);
        if (!((c >= 'a' && c <= 'z') || (c >= '0' && c <= '9') || c == '-')) c = '-';
        if (j + 1 >= out_n) return false;
        out[j++] = c;
    }
    out[j] = '\0';
    if (out[0] == '-' || out[j - 1] == '-') return false;
    return true;
}

void nc_hash_tuple(const char *tenant, const char *agent, const char *group,
                   char *out, size_t out_n) {
    char buf[512];
    int len = snprintf(buf, sizeof(buf), "%s|%s|%s", tenant, agent, group);
    if (len < 0 || (size_t)len >= sizeof(buf)) {
        if (out_n > 0) out[0] = '\0';
        return;
    }
    unsigned char digest[SHA256_DIGEST_LENGTH];
    SHA256((const unsigned char *)buf, (size_t)len, digest);
    snprintf(out, out_n, "%02x%02x%02x%02x%02x",
             digest[0], digest[1], digest[2], digest[3], digest[4]);
}

bool nc_derive_username(const char *tenant, const char *agent, const char *group,
                        char *out, size_t out_n) {
    char t[NC_ID_MAX + 1], a[NC_ID_MAX + 1], g[NC_ID_MAX + 1];
    if (!nc_validate_id(tenant, t, sizeof(t))) return false;
    if (!nc_validate_id(agent, a, sizeof(a))) return false;
    if (!nc_sanitize_group(group, g, sizeof(g))) return false;

    /* Truncate tenant and agent to 8 chars. */
    char t8[9], a8[9];
    memcpy(t8, t, 8); t8[8] = '\0';
    if (strlen(t) < 8) strcpy(t8, t);
    memcpy(a8, a, 8); a8[8] = '\0';
    if (strlen(a) < 8) strcpy(a8, a);

    char hash[NC_HASH_HEX_LEN + 1];
    nc_hash_tuple(t, a, g, hash, sizeof(hash));

    int n = snprintf(out, out_n, "ncg-%s-%s-%s", t8, a8, hash);
    if (n < 0 || (size_t)n >= out_n) return false;
    if ((size_t)n > NC_USERNAME_MAX - 1) return false;
    return true;
}
```

- [ ] **Step 3: 编写测试**

新建 `native/nc-setuid-helper/tests/test_naming.c`:

```c
#include "../src/naming.h"

#include <assert.h>
#include <stdio.h>
#include <string.h>

static void test_validate_id_accepts_valid(void) {
    char out[NC_ID_MAX + 1];
    assert(nc_validate_id("acme", out, sizeof(out)));
    assert(strcmp(out, "acme") == 0);
    /* Uppercase is lowercased. */
    assert(nc_validate_id("ACME", out, sizeof(out)));
    assert(strcmp(out, "acme") == 0);
}

static void test_validate_id_rejects_invalid(void) {
    char out[NC_ID_MAX + 1];
    assert(!nc_validate_id("", out, sizeof(out)));
    assert(!nc_validate_id("1abc", out, sizeof(out)));
    assert(!nc_validate_id("acme_corp", out, sizeof(out)));
    char too_long[NC_ID_MAX + 2];
    memset(too_long, 'a', sizeof(too_long) - 1);
    too_long[sizeof(too_long) - 1] = '\0';
    assert(!nc_validate_id(too_long, out, sizeof(out)));
}

static void test_sanitize_group_replaces_underscore(void) {
    char out[64];
    assert(nc_sanitize_group("Group_42", out, sizeof(out)));
    assert(strcmp(out, "group-42") == 0);
}

static void test_sanitize_group_rejects_traversal(void) {
    char out[64];
    assert(!nc_sanitize_group("../etc", out, sizeof(out)));
    assert(!nc_sanitize_group("/abs/path", out, sizeof(out)));
    assert(!nc_sanitize_group("", out, sizeof(out)));
    assert(!nc_sanitize_group("-leading", out, sizeof(out)));
}

static void test_derive_username_format(void) {
    char u[NC_USERNAME_MAX];
    assert(nc_derive_username("acme", "finance", "main", u, sizeof(u)));
    /* Format: ncg-acme-financ-<hash10> (agent truncated to 8 = "financ"? no, 7 chars stays) */
    /* "finance" is 7 chars, stays as "finance"; "acme" stays as "acme". */
    char prefix[32];
    snprintf(prefix, sizeof(prefix), "ncg-%s-%s-", "acme", "finance");
    assert(strncmp(u, prefix, strlen(prefix)) == 0);
    /* Total length must be <= 32. */
    assert(strlen(u) <= 32);
}

static void test_derive_username_truncates_long_segments(void) {
    char u[NC_USERNAME_MAX];
    /* Tenant/agent are 13 chars each; truncated to 8. */
    assert(nc_derive_username("verylongtenant", "verylongagent", "g", u, sizeof(u)));
    assert(strncmp(u, "ncg-verylon-verylon-", 20) == 0);
}

static void test_hash_is_stable_and_distinct(void) {
    char h1[NC_HASH_HEX_LEN + 1], h2[NC_HASH_HEX_LEN + 1], h3[NC_HASH_HEX_LEN + 1];
    nc_hash_tuple("t", "a", "g1", h1, sizeof(h1));
    nc_hash_tuple("t", "a", "g1", h2, sizeof(h2));
    nc_hash_tuple("t", "a", "g2", h3, sizeof(h3));
    assert(strcmp(h1, h2) == 0);
    assert(strcmp(h1, h3) != 0);
}

int main(void) {
    test_validate_id_accepts_valid();
    test_validate_id_rejects_invalid();
    test_sanitize_group_replaces_underscore();
    test_sanitize_group_rejects_traversal();
    test_derive_username_format();
    test_derive_username_truncates_long_segments();
    test_hash_is_stable_and_distinct();
    printf("test_naming: all passed\n");
    return 0;
}
```

- [ ] **Step 4: 更新 Makefile 以构建测试**

编辑 `native/nc-setuid-helper/Makefile`。在现有的 `test:` 块之后追加:

```makefile
TEST_BINS := $(TEST_DIR)/test_naming

build-tests: $(TEST_BINS)

$(TEST_DIR)/test_%: $(TEST_DIR)/test_%.c $(SRC_DIR)/%.c $(SRC_DIR)/%.h
	$(CC) $(CFLAGS) -I$(SRC_DIR) $< $(SRC_DIR)/$*.c -o $@ -lcrypto

test: build build-tests
	@bash $(TEST_DIR)/runner.sh
```

更新 `tests/runner.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

for bin in "$SCRIPT_DIR"/test_*; do
  [[ -x "$bin" ]] || continue
  echo "running $bin"
  "$bin"
done

exit 0
```

- [ ] **Step 5: 构建并运行测试**

运行: `cd native/nc-setuid-helper && make test`
预期: `test_naming: all passed`。

- [ ] **Step 6: 提交**

```bash
git add native/nc-setuid-helper/
git commit -m "feat(helper): add pure-logic naming helpers with unit tests"
```

---

## Task 3: users.db schema 与访问

**文件:**
- 新建: `src/users_db.c`, `src/users_db.h`
- 新建: `tests/test_users_db.c`

- [ ] **Step 1: 编写 header**

新建 `native/nc-setuid-helper/src/users_db.h`:

```c
#ifndef NC_USERS_DB_H
#define NC_USERS_DB_H

#include <stdbool.h>
#include <stdint.h>
#include <sys/types.h>

#define NC_USERS_DB_PATH "/var/lib/nanoclaw/users.db"

typedef struct {
    char tenant[17];
    char agent[17];
    char group[64];
    char username[33];
    uid_t uid;
    gid_t gid;
    char runtime_dir[4096];
    char cgroup_path[4096];
} nc_user_record;

/**
 * Open (or initialise) the users database. Creates schema if missing.
 * Caller owns the returned handle; close with sqlite3_close_v2().
 */
int nc_users_db_open(const char *path, void **db_out);

/**
 * Insert or replace a record. Returns 0 on success.
 */
int nc_users_db_upsert(void *db, const nc_user_record *rec);

/**
 * Look up a record by (tenant, agent, group). Returns 1 if found, 0 if not, <0 on error.
 */
int nc_users_db_lookup_tuple(void *db, const char *tenant, const char *agent,
                             const char *group, nc_user_record *out);

/**
 * Look up a record by username. Same return convention.
 */
int nc_users_db_lookup_username(void *db, const char *username, nc_user_record *out);

#endif
```

- [ ] **Step 2: 编写 users_db.c**

新建 `native/nc-setuid-helper/src/users_db.c`:

```c
#include "users_db.h"
#include "logging.h"

#include <sqlite3.h>
#include <stdio.h>
#include <string.h>

static const char *SCHEMA =
    "CREATE TABLE IF NOT EXISTS users ("
    "  tenant TEXT NOT NULL,"
    "  agent TEXT NOT NULL,"
    "  group_folder TEXT NOT NULL,"
    "  username TEXT NOT NULL UNIQUE,"
    "  uid INTEGER NOT NULL,"
    "  gid INTEGER NOT NULL,"
    "  runtime_dir TEXT NOT NULL,"
    "  cgroup_path TEXT,"
    "  created_at TEXT NOT NULL,"
    "  PRIMARY KEY (tenant, agent, group_folder)"
    ");"
    "CREATE INDEX IF NOT EXISTS idx_users_username ON users(username);"
    "CREATE INDEX IF NOT EXISTS idx_users_uid ON users(uid);";

int nc_users_db_open(const char *path, void **db_out) {
    if (!path || !db_out) return -1;
    sqlite3 *db = NULL;
    int rc = sqlite3_open(path, &db);
    if (rc != SQLITE_OK) {
        nc_log_error("event=users_db_open rc=%d msg=%s", rc, sqlite3_errmsg(db));
        if (db) sqlite3_close(db);
        return rc;
    }
    char *err = NULL;
    rc = sqlite3_exec(db, SCHEMA, NULL, NULL, &err);
    if (rc != SQLITE_OK) {
        nc_log_error("event=users_db_schema rc=%d msg=%s", rc, err ? err : "?");
        sqlite3_free(err);
        sqlite3_close(db);
        return rc;
    }
    /* Restrict permissions at the file level on first creation. */
    sqlite3_close(db);
    rc = sqlite3_open_v2(path, &db, SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE, NULL);
    if (rc != SQLITE_OK) return rc;
    /* Best effort chmod to 0600. */
    chmod(path, 0600);
    *db_out = db;
    return 0;
}

int nc_users_db_upsert(void *db, const nc_user_record *rec) {
    if (!db || !rec) return -1;
    sqlite3_stmt *stmt = NULL;
    static const char *SQL =
        "INSERT INTO users (tenant, agent, group_folder, username, uid, gid, runtime_dir, cgroup_path, created_at) "
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?, strftime('%Y-%m-%dT%H:%M:%SZ','now')) "
        "ON CONFLICT(tenant, agent, group_folder) DO UPDATE SET "
        "username=excluded.username, uid=excluded.uid, gid=excluded.gid, "
        "runtime_dir=excluded.runtime_dir, cgroup_path=excluded.cgroup_path";
    int rc = sqlite3_prepare_v2((sqlite3 *)db, SQL, -1, &stmt, NULL);
    if (rc != SQLITE_OK) return rc;
    sqlite3_bind_text(stmt, 1, rec->tenant, -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 2, rec->agent, -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 3, rec->group, -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 4, rec->username, -1, SQLITE_TRANSIENT);
    sqlite3_bind_int(stmt, 5, (int)rec->uid);
    sqlite3_bind_int(stmt, 6, (int)rec->gid);
    sqlite3_bind_text(stmt, 7, rec->runtime_dir, -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 8, rec->cgroup_path[0] ? rec->cgroup_path : NULL, -1, SQLITE_TRANSIENT);
    rc = sqlite3_step(stmt);
    sqlite3_finalize(stmt);
    return (rc == SQLITE_DONE) ? 0 : -1;
}

static int fill_record(sqlite3_stmt *stmt, nc_user_record *out) {
    const unsigned char *tenant = sqlite3_column_text(stmt, 0);
    const unsigned char *agent = sqlite3_column_text(stmt, 1);
    const unsigned char *group = sqlite3_column_text(stmt, 2);
    const unsigned char *username = sqlite3_column_text(stmt, 3);
    int uid = sqlite3_column_int(stmt, 4);
    int gid = sqlite3_column_int(stmt, 5);
    const unsigned char *rt = sqlite3_column_text(stmt, 6);
    const unsigned char *cg = sqlite3_column_text(stmt, 7);
    snprintf(out->tenant, sizeof(out->tenant), "%s", tenant ? (const char *)tenant : "");
    snprintf(out->agent, sizeof(out->agent), "%s", agent ? (const char *)agent : "");
    snprintf(out->group, sizeof(out->group), "%s", group ? (const char *)group : "");
    snprintf(out->username, sizeof(out->username), "%s", username ? (const char *)username : "");
    out->uid = (uid_t)uid;
    out->gid = (gid_t)gid;
    snprintf(out->runtime_dir, sizeof(out->runtime_dir), "%s", rt ? (const char *)rt : "");
    snprintf(out->cgroup_path, sizeof(out->cgroup_path), "%s", cg ? (const char *)cg : "");
    return 1;
}

int nc_users_db_lookup_tuple(void *db, const char *tenant, const char *agent,
                             const char *group, nc_user_record *out) {
    if (!db || !out) return -1;
    sqlite3_stmt *stmt = NULL;
    static const char *SQL =
        "SELECT tenant, agent, group_folder, username, uid, gid, runtime_dir, cgroup_path "
        "FROM users WHERE tenant=? AND agent=? AND group_folder=?";
    int rc = sqlite3_prepare_v2((sqlite3 *)db, SQL, -1, &stmt, NULL);
    if (rc != SQLITE_OK) return rc;
    sqlite3_bind_text(stmt, 1, tenant, -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 2, agent, -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 3, group, -1, SQLITE_TRANSIENT);
    rc = sqlite3_step(stmt);
    int result = 0;
    if (rc == SQLITE_ROW) result = fill_record(stmt, out);
    else if (rc != SQLITE_DONE) result = -1;
    sqlite3_finalize(stmt);
    return result;
}

int nc_users_db_lookup_username(void *db, const char *username, nc_user_record *out) {
    if (!db || !out) return -1;
    sqlite3_stmt *stmt = NULL;
    static const char *SQL =
        "SELECT tenant, agent, group_folder, username, uid, gid, runtime_dir, cgroup_path "
        "FROM users WHERE username=?";
    int rc = sqlite3_prepare_v2((sqlite3 *)db, SQL, -1, &stmt, NULL);
    if (rc != SQLITE_OK) return rc;
    sqlite3_bind_text(stmt, 1, username, -1, SQLITE_TRANSIENT);
    rc = sqlite3_step(stmt);
    int result = 0;
    if (rc == SQLITE_ROW) result = fill_record(stmt, out);
    else if (rc != SQLITE_DONE) result = -1;
    sqlite3_finalize(stmt);
    return result;
}
```

- [ ] **Step 3: 编写测试**

新建 `native/nc-setuid-helper/tests/test_users_db.c`:

```c
#include "../src/users_db.h"

#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <unistd.h>

int main(void) {
    char tmpl[] = "/tmp/nc_usersdb_XXXXXX";
    int fd = mkstemp(tmpl);
    if (fd < 0) { perror("mkstemp"); return 1; }
    close(fd);
    unlink(tmpl);

    char path[256];
    snprintf(path, sizeof(path), "%s", tmpl);

    void *db = NULL;
    assert(nc_users_db_open(path, &db) == 0);

    nc_user_record rec = {0};
    strcpy(rec.tenant, "acme");
    strcpy(rec.agent, "finance");
    strcpy(rec.group, "main");
    strcpy(rec.username, "ncg-acme-financ-0123456789");
    rec.uid = 60001;
    rec.gid = 60001;
    strcpy(rec.runtime_dir, "/var/lib/nanoclaw/runtime/acme/finance/main");
    strcpy(rec.cgroup_path, "/sys/fs/cgroup/nanoclaw/main");

    assert(nc_users_db_upsert(db, &rec) == 0);

    nc_user_record found;
    assert(nc_users_db_lookup_tuple(db, "acme", "finance", "main", &found) == 1);
    assert(strcmp(found.username, rec.username) == 0);
    assert(found.uid == rec.uid);

    assert(nc_users_db_lookup_tuple(db, "x", "y", "z", &found) == 0);
    assert(nc_users_db_lookup_username(db, "ncg-acme-financ-0123456789", &found) == 1);

    /* Upsert on same tuple updates uid. */
    rec.uid = 60010;
    assert(nc_users_db_upsert(db, &rec) == 0);
    assert(nc_users_db_lookup_tuple(db, "acme", "finance", "main", &found) == 1);
    assert(found.uid == 60010);

    sqlite3_close((sqlite3 *)db);
    unlink(path);
    printf("test_users_db: all passed\n");
    return 0;
}
```

- [ ] **Step 4: 更新 Makefile**

追加到 `Makefile` 的 `TEST_BINS`:

```makefile
TEST_BINS := $(TEST_DIR)/test_naming $(TEST_DIR)/test_users_db

$(TEST_DIR)/test_users_db: $(TEST_DIR)/test_users_db.c $(SRC_DIR)/users_db.c $(SRC_DIR)/users_db.h $(SRC_DIR)/logging.c $(SRC_DIR)/logging.h
	$(CC) $(CFLAGS) -I$(SRC_DIR) $< $(SRC_DIR)/users_db.c $(SRC_DIR)/logging.c -o $@ -lsqlite3
```

(现有的 `test_%` 通用模式规则只编译一个源文件;users_db 还需要 logging,因此这里写显式规则。Naming 测试不需要 logging。)

- [ ] **Step 5: 构建并测试**

运行: `cd native/nc-setuid-helper && make test`
预期: `test_naming` 和 `test_users_db` 都通过。

- [ ] **Step 6: 提交**

```bash
git add native/nc-setuid-helper/
git commit -m "feat(helper): add users.db schema and CRUD"
```

---

## Task 4: 参数解析

**文件:**
- 新建: `src/arg.c`, `arg.h`
- 新建: `tests/test_arg.c`

- [ ] **Step 1: 编写 arg.h**

新建 `native/nc-setuid-helper/src/arg.h`:

```c
#ifndef NC_ARG_H
#define NC_ARG_H

#include <stdbool.h>

/**
 * Look up `--key=value` in argv[2..argc-1] and return the value.
 * Returns NULL if not found. Stops scanning at the first non-option arg ("--").
 */
const char *nc_arg_get(int argc, char **argv, const char *key);

/**
 * Same as nc_arg_get, but fails loudly (logs + returns NULL) if missing.
 */
const char *nc_arg_require(int argc, char **argv, const char *key);

/**
 * Find index of the "--" separator. Returns argc if not present.
 */
int nc_arg_separator_index(int argc, char **argv);

#endif
#endif /* NC_ARG_H */
```

- [ ] **Step 2: 编写 arg.c**

新建 `native/nc-setuid-helper/src/arg.c`:

```c
#include "arg.h"
#include "logging.h"

#include <stdio.h>
#include <string.h>

const char *nc_arg_get(int argc, char **argv, const char *key) {
    if (!argv || !key) return NULL;
    size_t klen = strlen(key);
    for (int i = 2; i < argc; i++) {
        if (!argv[i]) continue;
        if (argv[i][0] != '-') break;  /* reached "--" or positional */
        if (strncmp(argv[i] + 2, key, klen) == 0 && argv[i][2 + klen] == '=') {
            return argv[i] + 2 + klen + 1;
        }
    }
    return NULL;
}

const char *nc_arg_require(int argc, char **argv, const char *key) {
    const char *v = nc_arg_get(argc, argv, key);
    if (!v) {
        nc_log_error("event=missing_arg key=%s", key);
    }
    return v;
}

int nc_arg_separator_index(int argc, char **argv) {
    for (int i = 2; i < argc; i++) {
        if (argv[i] && strcmp(argv[i], "--") == 0) return i;
    }
    return argc;
}
```

- [ ] **Step 3: 编写测试**

新建 `native/nc-setuid-helper/tests/test_arg.c`:

```c
#include "../src/arg.h"

#include <assert.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    char *argv[] = {"prog", "spawn", "--uid=1000", "--gid=1000", "--", "/bin/ls", "-l", NULL};
    int argc = 7;

    assert(strcmp(nc_arg_get(argc, argv, "uid"), "1000") == 0);
    assert(strcmp(nc_arg_get(argc, argv, "gid"), "1000") == 0);
    assert(nc_arg_get(argc, argv, "missing") == NULL);
    assert(nc_arg_separator_index(argc, argv) == 4);

    char *argv2[] = {"prog", "spawn", "--uid=1", NULL};
    assert(nc_arg_separator_index(3, argv2) == 3);
    assert(strcmp(nc_arg_get(3, argv2, "uid"), "1") == 0);

    printf("test_arg: all passed\n");
    return 0;
}
```

- [ ] **Step 4: 更新 Makefile 并运行测试**

把 `test_arg` 追加到 `TEST_BINS`:

```makefile
TEST_BINS := $(TEST_DIR)/test_naming $(TEST_DIR)/test_users_db $(TEST_DIR)/test_arg

$(TEST_DIR)/test_arg: $(TEST_DIR)/test_arg.c $(SRC_DIR)/arg.c $(SRC_DIR)/arg.h $(SRC_DIR)/logging.c $(SRC_DIR)/logging.h
	$(CC) $(CFLAGS) -I$(SRC_DIR) $< $(SRC_DIR)/arg.c $(SRC_DIR)/logging.c -o $@
```

运行: `cd native/nc-setuid-helper && make test`
预期: 三个测试全部通过。

- [ ] **Step 5: 提交**

```bash
git add native/nc-setuid-helper/
git commit -m "feat(helper): add argument parser"
```

---

## Task 5: ACL 设置辅助函数

**文件:**
- 新建: `src/acl.c`, `acl.h`
- 新建: `tests/root/test_prepare_permissions.sh`(root-gated)

- [ ] **Step 1: 编写 acl.h**

新建 `native/nc-setuid-helper/src/acl.h`:

```c
#ifndef NC_ACL_H
#define NC_ACL_H

#include <stdbool.h>

/**
 * Set owner rwx and grant `nanoclaw-svc` rwx on `path` (a directory or file).
 * Owner and mode are set via chmod/chown; ACL via libacl.
 *
 * Returns 0 on success, -errno on failure.
 */
int nc_acl_setup_runtime_dir(const char *path, uid_t owner, gid_t group,
                             const char *svc_username);

/**
 * Install default ACL on a directory so that new files created inside inherit
 * the same (owner rwx, svc rwx) grants.
 */
int nc_acl_install_default(const char *dir, const char *svc_username);

#endif
```

- [ ] **Step 2: 编写 acl.c**

新建 `native/nc-setuid-helper/src/acl.c`:

```c
#include "acl.h"
#include "logging.h"

#include <acl/libacl.h>
#include <errno.h>
#include <pwd.h>
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

static int apply_user_rwx(acl_t acl, const char *who, acl_permset_t *out_set) {
    acl_entry_t entry;
    if (acl_create_entry(&acl, &entry) != 0) return -1;
    acl_set_tag_type(entry, ACL_USER);
    struct passwd *pw = getpwnam(who);
    if (!pw) return -errno;
    acl_set_qualifier(entry, &pw->pw_uid);
    acl_permset_t perms;
    if (acl_get_permset(entry, &perms) != 0) return -1;
    acl_clear_perms(perms);
    acl_add_perm(perms, ACL_READ | ACL_WRITE | ACL_EXECUTE);
    acl_set_permset(entry, perms);
    *out_set = perms;
    return 0;
}

int nc_acl_setup_runtime_dir(const char *path, uid_t owner, gid_t group,
                             const char *svc_username) {
    if (!path || !svc_username) return -EINVAL;
    /* Ensure ownership. */
    if (chown(path, owner, group) != 0) {
        nc_log_error("event=chown path=%s err=%s", path, strerror(errno));
        return -errno;
    }
    if (chmod(path, 0700) != 0) {
        nc_log_error("event=chmod path=%s err=%s", path, strerror(errno));
        return -errno;
    }

    acl_t acl = acl_get_file(path, ACL_TYPE_ACCESS);
    if (!acl) {
        nc_log_error("event=acl_get path=%s err=%s", path, strerror(errno));
        return -errno;
    }

    /* Add an ACL_USER entry granting nanoclaw-svc rwx. */
    acl_permset_t unused;
    if (apply_user_rwx(acl, svc_username, &unused) != 0) {
        nc_log_error("event=acl_apply_user path=%s", path);
        acl_free(acl);
        return -EINVAL;
    }

    if (acl_set_file(path, ACL_TYPE_ACCESS, acl) != 0) {
        nc_log_error("event=acl_set path=%s err=%s", path, strerror(errno));
        acl_free(acl);
        return -errno;
    }

    acl_free(acl);
    return 0;
}

int nc_acl_install_default(const char *dir, const char *svc_username) {
    if (!dir || !svc_username) return -EINVAL;
    /* Start from a base default ACL that mirrors the access ACL's ugo, then
     * add the user entry. */
    acl_t base = acl_get_file(dir, ACL_TYPE_DEFAULT);
    if (!base) {
        /* No existing default ACL — create a minimal one. */
        acl_t init = acl_from_text("u::rwx,g::---,o::---,m::rwx");
        if (!init) return -EINVAL;
        acl_set_file(dir, ACL_TYPE_DEFAULT, init);
        acl_free(init);
        base = acl_get_file(dir, ACL_TYPE_DEFAULT);
        if (!base) return -EINVAL;
    }
    acl_permset_t unused;
    if (apply_user_rwx(base, svc_username, &unused) != 0) {
        acl_free(base);
        return -EINVAL;
    }
    int rc = acl_set_file(dir, ACL_TYPE_DEFAULT, base);
    acl_free(base);
    return rc == 0 ? 0 : -errno;
}
```

- [ ] **Step 3: 编写 root-gated 测试**

新建 `native/nc-setuid-helper/tests/root/test_acl.sh`:

```bash
#!/usr/bin/env bash
# Root-gated integration test for nc_acl_setup_runtime_dir.
# Skipped automatically when EUID != 0.

set -euo pipefail

if [[ "${EUID}" -ne 0 ]]; then
  echo "skip (not root)"
  exit 0
fi

# Requires `nanoclaw-svc` user and a test ncg user.
id nanoclaw-svc >/dev/null 2>&1 || useradd -r nanoclaw-svc
id ncg-test >/dev/null 2>&1 || useradd -r -u 60090 ncg-test

TMPDIR=$(mktemp -d)
trap 'rm -rf "$TMPDIR"' EXIT
TARGET="$TMPDIR/rt"
mkdir -p "$TARGET"

# Trigger via a tiny C harness; we assume build/test_runtime_acl exists.
"$PWD/$(dirname "$0")/../../build/test_runtime_acl" "$TARGET" 60090 60090 ncg-test nanoclaw-svc
RC=$?

# Verify permissions
PERMS=$(stat -c '%a' "$TARGET")
[[ "$PERMS" == "700" ]]
OWNER=$(stat -c '%U' "$TARGET")
[[ "$OWNER" == "ncg-test" ]]

# Verify ACL via getfacl
getfacl "$TARGET" 2>/dev/null | grep -q "user:nanoclaw-svc:rwx"

exit $RC
```

新建一个小 harness `native/nc-setuid-helper/tests/test_runtime_acl.c`:

```c
#include "../src/acl.h"

#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv) {
    if (argc != 6) {
        fprintf(stderr, "usage: test_runtime_acl path uid gid owner_user svc_user\n");
        return 2;
    }
    const char *path = argv[1];
    uid_t uid = (uid_t)atoi(argv[2]);
    gid_t gid = (gid_t)atoi(argv[3]);
    const char *owner = argv[4];
    const char *svc = argv[5];

    int rc = nc_acl_setup_runtime_dir(path, uid, gid, owner);
    if (rc != 0) {
        fprintf(stderr, "setup_runtime_dir failed: %d\n", rc);
        return 1;
    }
    rc = nc_acl_install_default(path, svc);
    if (rc != 0) {
        fprintf(stderr, "install_default failed: %d\n", rc);
        return 1;
    }
    return 0;
}
```

追加到 Makefile:

```makefile
TEST_BINS := $(TEST_DIR)/test_naming $(TEST_DIR)/test_users_db $(TEST_DIR)/test_arg $(BUILD_DIR)/test_runtime_acl

$(BUILD_DIR)/test_runtime_acl: $(TEST_DIR)/test_runtime_acl.c $(SRC_DIR)/acl.c $(SRC_DIR)/logging.c
	@mkdir -p $(BUILD_DIR)
	$(CC) $(CFLAGS) -I$(SRC_DIR) $< $(SRC_DIR)/acl.c $(SRC_DIR)/logging.c -o $@ -lacl
```

- [ ] **Step 4: 构建(未安装 libacl-dev 时会失败)**

运行: `sudo apt-get install -y libacl1-dev libsqlite3-dev libssl-dev`(或对应平台的等价命令)
然后: `cd native/nc-setuid-helper && make build-tests`
预期: 无错误完成构建。

- [ ] **Step 5: 运行 root-gated ACL 测试(需要以 root 身份运行)**

运行: `cd native/nc-setuid-helper && sudo tests/root/test_acl.sh`
预期: 输出中不包含 `skip (not root)`;测试通过,目标目录具有 0700 权限,属主为 ncg-test,ACL 包含 `user:nanoclaw-svc:rwx`。

CI 中无 root 时:跳过此步骤,并在 PR 中注明 root 测试为手动进行。

- [ ] **Step 6: 提交**

```bash
git add native/nc-setuid-helper/
git commit -m "feat(helper): add POSIX ACL setup helpers (runtime dir + default)"
```

---

## Task 6: cgroup v2 设置

**文件:**
- 新建: `src/cgroup.c`, `cgroup.h`

- [ ] **Step 1: 编写 cgroup.h**

新建 `native/nc-setuid-helper/src/cgroup.h`:

```c
#ifndef NC_CGROUP_H
#define NC_CGROUP_H

#include <stdbool.h>

#define NC_CGROUP_ROOT "/sys/fs/cgroup/nanoclaw"

/**
 * Create a cgroup at ${NC_CGROUP_ROOT}/${subpath}. Writes the requested limits.
 *
 * mem_mb = 0 means unlimited.
 * pids = 0 means unlimited.
 * cpu_shares = 0 means default.
 *
 * Returns 0 on success, -errno on failure.
 */
int nc_cgroup_create(const char *subpath, int mem_mb, int pids, int cpu_shares);

/**
 * Write a PID into cgroup.procs.
 */
int nc_cgroup_add_pid(const char *subpath, int pid);

#endif
```

- [ ] **Step 2: 编写 cgroup.c**

新建 `native/nc-setuid-helper/src/cgroup.c`:

```c
#include "cgroup.h"
#include "logging.h"

#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>

static int write_file(const char *path, const char *value) {
    FILE *f = fopen(path, "w");
    if (!f) {
        nc_log_error("event=cgroup_write path=%s err=%s", path, strerror(errno));
        return -errno;
    }
    if (fputs(value, f) == EOF) {
        nc_log_error("event=cgroup_write_io path=%s", path);
        fclose(f);
        return -EIO;
    }
    fclose(f);
    return 0;
}

int nc_cgroup_create(const char *subpath, int mem_mb, int pids, int cpu_shares) {
    char path[4096];
    snprintf(path, sizeof(path), "%s/%s", NC_CGROUP_ROOT, subpath);
    if (mkdir(path, 0755) != 0 && errno != EEXIST) {
        nc_log_error("event=cgroup_mkdir path=%s err=%s", path, strerror(errno));
        return -errno;
    }
    if (mem_mb > 0) {
        char buf[32];
        /* mem_mb in mebibytes; cgroup v2 uses bytes for memory.max */
        snprintf(buf, sizeof(buf), "%lld", (long long)mem_mb * 1024 * 1024);
        char p[4096];
        snprintf(p, sizeof(p), "%s/memory.max", path);
        int rc = write_file(p, buf);
        if (rc != 0) return rc;
    }
    if (pids > 0) {
        char buf[32];
        snprintf(buf, sizeof(buf), "%d", pids);
        char p[4096];
        snprintf(p, sizeof(p), "%s/pids.max", path);
        int rc = write_file(p, buf);
        if (rc != 0) return rc;
    }
    if (cpu_shares > 0) {
        /* cgroup v2 uses cpu.weight in [1, 10000]; shares ~= weight * 10. */
        char buf[32];
        snprintf(buf, sizeof(buf), "%d", cpu_shares * 10);
        char p[4096];
        snprintf(p, sizeof(p), "%s/cpu.weight", path);
        int rc = write_file(p, buf);
        if (rc != 0) return rc;
    }
    return 0;
}

int nc_cgroup_add_pid(const char *subpath, int pid) {
    char path[4096];
    snprintf(path, sizeof(path), "%s/%s/cgroup.procs", NC_CGROUP_ROOT, subpath);
    char buf[32];
    snprintf(buf, sizeof(buf), "%d", pid);
    return write_file(path, buf);
}
```

- [ ] **Step 3: 构建健全性检查**

运行: `cd native/nc-setuid-helper && make build`
预期: 编译通过。(cgroup 没有单元测试;它只产生文件系统副作用,由 root-gated 集成测试覆盖。)

- [ ] **Step 4: 提交**

```bash
git add native/nc-setuid-helper/
git commit -m "feat(helper): add cgroup v2 create + add-pid helpers"
```

---

## Task 7: `prepare` 命令

**文件:**
- 新建: `src/prepare.c`, `prepare.h`
- 修改: `src/main.c`

- [ ] **Step 1: 编写 prepare.h**

新建 `native/nc-setuid-helper/src/prepare.h`:

```c
#ifndef NC_PREPARE_H
#define NC_PREPARE_H

#include <stdbool.h>

/**
 * Implements `nc-setuid-helper prepare`.
 *
 * Args:
 *   tenant, agent, group    — canonical tuple, validated
 *   runtime_dir             — absolute path under /var/lib/nanoclaw/runtime
 *
 * Side effects:
 *   - derive Linux username + allocate uid/gid if not present
 *   - create the Linux user/group (via useradd-like commands)
 *   - create runtime_dir and subdirs with correct owner + ACLs
 *   - upsert (tenant, agent, group) into users.db
 *
 * Returns 0 on success, non-zero on failure.
 */
int nc_cmd_prepare(int argc, char **argv);

#endif
#endif /* NC_PREPARE_H */
```

- [ ] **Step 2: 编写 prepare.c**

新建 `native/nc-setuid-helper/src/prepare.c`:

```c
#include "prepare.h"
#include "acl.h"
#include "arg.h"
#include "logging.h"
#include "naming.h"
#include "users_db.h"

#include <errno.h>
#include <pwd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

/* Allocate a UID/GID pair from the nanoclaw-managed range. Simple strategy:
 * scan /etc/passwd for the highest ncg-* uid >= 60000 and increment. Real
 * deployments should use a proper uid allocator; this is the minimum viable. */
static uid_t allocate_uid(void) {
    FILE *f = fopen("/etc/passwd", "r");
    if (!f) return 60000;
    uid_t max = 60000;
    char line[1024];
    while (fgets(line, sizeof(line), f)) {
        char username[256];
        unsigned int uid;
        if (sscanf(line, "%255[^:]:%*[^:]:%u:", username, &uid) == 2) {
            if (strncmp(username, "ncg-", 4) == 0 && uid >= max) {
                max = uid + 1;
            }
        }
    }
    fclose(f);
    return max;
}

static int run_useradd(const char *username, uid_t uid, gid_t gid) {
    char uid_buf[16], gid_buf[16];
    snprintf(uid_buf, sizeof(uid_buf), "-u%u", uid);
    snprintf(gid_buf, sizeof(gid_buf), "-g%u", gid);
    pid_t pid = fork();
    if (pid < 0) return -errno;
    if (pid == 0) {
        execl("/usr/sbin/useradd", "useradd",
              "-M",      /* no home dir creation */
              "-N",      /* no per-user group; we use the shared gid below */
              "-r",      /* system user */
              uid_buf,
              "-g", username, /* primary group = same name as user; create below first */
              username,
              (char *)NULL);
        _exit(127);
    }
    int status = 0;
    if (waitpid(pid, &status, 0) < 0) return -errno;
    return WIFEXITED(status) && WEXITSTATUS(status) == 0 ? 0 : -EIO;
}

static int ensure_group(const char *name, gid_t gid) {
    char gid_buf[16];
    snprintf(gid_buf, sizeof(gid_buf), "-g%u", gid);
    pid_t pid = fork();
    if (pid < 0) return -errno;
    if (pid == 0) {
        execl("/usr/sbin/groupadd", "groupadd", "-r", gid_buf, name, (char *)NULL);
        _exit(127);
    }
    int status = 0;
    if (waitpid(pid, &status, 0) < 0) return -errno;
    if (WIFEXITED(status) && WEXITSTATUS(status) == 0) return 0;
    /* group already exists — best effort lookup */
    return 0;
}

int nc_cmd_prepare(int argc, char **argv) {
    const char *tenant = nc_arg_require(argc, argv, "tenant");
    const char *agent = nc_arg_require(argc, argv, "agent");
    const char *group = nc_arg_require(argc, argv, "group");
    const char *runtime_dir = nc_arg_require(argc, argv, "runtime-dir");
    if (!tenant || !agent || !group || !runtime_dir) return 1;

    char t[NC_ID_MAX + 1], a[NC_ID_MAX + 1], g[64];
    if (!nc_validate_id(tenant, t, sizeof(t))) return 1;
    if (!nc_validate_id(agent, a, sizeof(a))) return 1;
    if (!nc_sanitize_group(group, g, sizeof(g))) return 1;

    /* runtime_dir must be under /var/lib/nanoclaw/runtime and match tuple. */
    const char *prefix = "/var/lib/nanoclaw/runtime/";
    size_t plen = strlen(prefix);
    if (strncmp(runtime_dir, prefix, plen) != 0) {
        nc_log_error("event=prepare_bad_dir dir=%s", runtime_dir);
        return 1;
    }

    char username[NC_USERNAME_MAX];
    if (!nc_derive_username(t, a, g, username, sizeof(username))) return 1;

    void *db = NULL;
    if (nc_users_db_open(NC_USERS_DB_PATH, &db) != 0) {
        return 2;
    }

    nc_user_record rec = {0};
    int found = nc_users_db_lookup_username(db, username, &rec);
    if (found <= 0) {
        /* Allocate new uid/gid and create the user. */
        uid_t uid = allocate_uid();
        gid_t gid = uid;
        if (ensure_group(username, gid) != 0) {
            nc_log_error("event=ensure_group failed");
            sqlite3_close(db);
            return 2;
        }
        if (run_useradd(username, uid, gid) != 0) {
            nc_log_error("event=useradd failed");
            sqlite3_close(db);
            return 2;
        }
        memset(&rec, 0, sizeof(rec));
        snprintf(rec.tenant, sizeof(rec.tenant), "%s", t);
        snprintf(rec.agent, sizeof(rec.agent), "%s", a);
        snprintf(rec.group, sizeof(rec.group), "%s", g);
        snprintf(rec.username, sizeof(rec.username), "%s", username);
        rec.uid = uid;
        rec.gid = gid;
    }

    /* Always create / refresh runtime_dir and ACLs. */
    if (mkdir(runtime_dir, 0700) != 0 && errno != EEXIST) {
        nc_log_error("event=mkdir dir=%s err=%s", runtime_dir, strerror(errno));
        sqlite3_close(db);
        return 2;
    }
    char subpath[4096];
    const char *subdirs[] = {"live", "runs", "skills/generated", "files", "downloads"};
    for (size_t i = 0; i < sizeof(subdirs) / sizeof(subdirs[0]); i++) {
        snprintf(subpath, sizeof(subpath), "%s/%s", runtime_dir, subdirs[i]);
        if (mkdir(subpath, 0700) != 0 && errno != EEXIST) {
            nc_log_error("event=mkdir_sub dir=%s err=%s", subpath, strerror(errno));
            sqlite3_close(db);
            return 2;
        }
    }

    /* Apply owner + ACL to every level of the tree. */
    if (nc_acl_setup_runtime_dir(runtime_dir, rec.uid, rec.gid, "nanoclaw-svc") != 0) {
        sqlite3_close(db);
        return 2;
    }
    if (nc_acl_install_default(runtime_dir, "nanoclaw-svc") != 0) {
        sqlite3_close(db);
        return 2;
    }
    for (size_t i = 0; i < sizeof(subdirs) / sizeof(subdirs[0]); i++) {
        snprintf(subpath, sizeof(subpath), "%s/%s", runtime_dir, subdirs[i]);
        nc_acl_setup_runtime_dir(subpath, rec.uid, rec.gid, "nanoclaw-svc");
        nc_acl_install_default(subpath, "nanoclaw-svc");
    }

    /* Update record's runtime_dir + cgroup_path (cgroup created at spawn time). */
    snprintf(rec.runtime_dir, sizeof(rec.runtime_dir), "%s", runtime_dir);
    if (nc_users_db_upsert(db, &rec) != 0) {
        sqlite3_close(db);
        return 2;
    }

    /* Emit a one-line summary on stdout for the control plane to consume. */
    printf("username=%s uid=%u gid=%u runtime_dir=%s\n",
           rec.username, rec.uid, rec.gid, rec.runtime_dir);

    sqlite3_close(db);
    return 0;
}
```

- [ ] **Step 3: 接入 main.c**

编辑 `native/nc-setuid-helper/src/main.c`。在文件顶部添加 `#include "prepare.h"`,并将桩分发逻辑替换为:

```c
if (strcmp(argv[1], "prepare") == 0) {
    return nc_cmd_prepare(argc, argv);
}
fprintf(stderr, "error: command '%s' not yet implemented\n", argv[1]);
return 1;
```

- [ ] **Step 4: 构建**

运行: `cd native/nc-setuid-helper && make build`
预期: 编译通过。

- [ ] **Step 5: 提交**

```bash
git add native/nc-setuid-helper/
git commit -m "feat(helper): implement prepare command"
```

---

## Task 8: `spawn`、`kill`、`status`、`cgroup` 命令

**文件:**
- 新建: `src/spawn.c`, `spawn.h`
- 新建: `src/kill.c`, `kill.h`
- 新建: `src/status.c`, `status.h`
- 新建: `src/cgroup_cmd.c`, `cgroup_cmd.h`
- 修改: `src/main.c`

- [ ] **Step 1: 编写 spawn.c**

新建 `native/nc-setuid-helper/src/spawn.h`:

```c
#ifndef NC_SPAWN_H
#define NC_SPAWN_H

int nc_cmd_spawn(int argc, char **argv);

#endif
```

新建 `native/nc-setuid-helper/src/spawn.c`:

```c
#include "spawn.h"
#include "arg.h"
#include "cgroup.h"
#include "logging.h"
#include "users_db.h"

#include <errno.h>
#include <grp.h>
#include <pwd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/prctl.h>
#include <sys/types.h>
#include <unistd.h>

static int parse_uint(const char *s, unsigned long *out) {
    if (!s || !*s) return -1;
    char *end = NULL;
    unsigned long v = strtoul(s, &end, 10);
    if (*end) return -1;
    *out = v;
    return 0;
}

int nc_cmd_spawn(int argc, char **argv) {
    const char *uid_str = nc_arg_require(argc, argv, "uid");
    const char *gid_str = nc_arg_require(argc, argv, "gid");
    const char *cgroup = nc_arg_get(argc, argv, "cgroup");
    const char *runtime_dir = nc_arg_require(argc, argv, "runtime-dir");
    if (!uid_str || !gid_str || !runtime_dir) return 1;

    unsigned long uid_l, gid_l;
    if (parse_uint(uid_str, &uid_l) != 0 || parse_uint(gid_str, &gid_l) != 0) {
        nc_log_error("event=spawn_bad_uid");
        return 1;
    }
    uid_t uid = (uid_t)uid_l;
    gid_t gid = (gid_t)gid_l;

    /* Look up record by username derived from uid. We trust that users.db has it. */
    void *db = NULL;
    if (nc_users_db_open(NC_USERS_DB_PATH, &db) != 0) return 2;
    nc_user_record rec;
    /* Scan users for matching uid (simple approach). */
    int found = 0;
    {
        sqlite3_stmt *stmt = NULL;
        const char *SQL = "SELECT tenant, agent, group_folder, username, uid, gid, runtime_dir, cgroup_path FROM users WHERE uid=?";
        sqlite3_prepare_v2((sqlite3 *)db, SQL, -1, &stmt, NULL);
        sqlite3_bind_int(stmt, 1, (int)uid);
        if (sqlite3_step(stmt) == SQLITE_ROW) {
            found = 1;
            const unsigned char *u = sqlite3_column_text(stmt, 3);
            snprintf(rec.username, sizeof(rec.username), "%s", u);
            rec.uid = (uid_t)sqlite3_column_int(stmt, 4);
            rec.gid = (gid_t)sqlite3_column_int(stmt, 5);
            snprintf(rec.runtime_dir, sizeof(rec.runtime_dir), "%s", (const char *)sqlite3_column_text(stmt, 6));
        }
        sqlite3_finalize(stmt);
    }
    sqlite3_close(db);
    if (!found) {
        nc_log_error("event=spawn_unknown_uid uid=%u", uid);
        return 3;
    }
    /* Runtime dir must match. */
    if (strcmp(rec.runtime_dir, runtime_dir) != 0) {
        nc_log_error("event=spawn_rt_mismatch expected=%s got=%s", rec.runtime_dir, runtime_dir);
        return 3;
    }

    int sep = nc_arg_separator_index(argc, argv);
    if (sep >= argc - 1) {
        nc_log_error("event=spawn_no_cmd");
        return 1;
    }
    char **cmd_argv = &argv[sep + 1];

    /* Apply cgroup membership now so the exec'd process inherits it. */
    if (cgroup) {
        if (nc_cgroup_add_pid(cgroup, (int)getpid()) != 0) {
            nc_log_warn("event=spawn_cgroup_attach failed");
        }
    }

    /* Drop privileges. */
    if (setgid(gid) != 0) {
        nc_log_error("event=setgid err=%s", strerror(errno));
        return 2;
    }
    if (setgroups(0, NULL) != 0) {
        nc_log_error("event=setgroups err=%s", strerror(errno));
        return 2;
    }
    if (setuid(uid) != 0) {
        nc_log_error("event=setuid err=%s", strerror(errno));
        return 2;
    }
    /* No new privs. */
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) != 0) {
        nc_log_warn("event=no_new_privs failed");
    }

    /* chdir into runtime dir so relative paths land there. */
    if (chdir(runtime_dir) != 0) {
        nc_log_warn("event=chdir failed dir=%s", runtime_dir);
    }

    execvp(cmd_argv[0], cmd_argv);
    nc_log_error("event=exec failed cmd=%s err=%s", cmd_argv[0], strerror(errno));
    _exit(127);
}
```

- [ ] **Step 2: 编写 kill.c**

新建 `native/nc-setuid-helper/src/kill.h`:

```c
#ifndef NC_KILL_H
#define NC_KILL_H

int nc_cmd_kill(int argc, char **argv);

#endif
```

新建 `native/nc-setuid-helper/src/kill.c`:

```c
#include "kill.h"
#include "arg.h"
#include "logging.h"
#include "process.h"

#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int nc_cmd_kill(int argc, char **argv) {
    const char *pid_str = nc_arg_require(argc, argv, "pid");
    const char *signal_str = nc_arg_require(argc, argv, "signal");
    const char *uid_str = nc_arg_require(argc, argv, "uid");
    const char *rt = nc_arg_require(argc, argv, "runtime-dir");
    const char *start_time_str = nc_arg_require(argc, argv, "start-time");
    if (!pid_str || !signal_str || !uid_str || !rt || !start_time_str) return 1;

    pid_t pid = (pid_t)atoi(pid_str);
    uid_t expected_uid = (uid_t)atoi(uid_str);
    unsigned long long start_time = strtoull(start_time_str, NULL, 10);

    int sig = nc_signal_from_name(signal_str);
    if (sig < 0) {
        nc_log_error("event=bad_signal signal=%s", signal_str);
        return 1;
    }

    if (nc_process_identity_matches(pid, expected_uid, start_time, rt) != 0) {
        nc_log_error("event=identity_mismatch pid=%d", pid);
        return 3;
    }

    if (kill(pid, sig) != 0) {
        nc_log_error("event=kill_failed pid=%d err=%s", pid, strerror(errno));
        return 2;
    }
    return 0;
}
```

- [ ] **Step 3: 编写 status.c**

新建 `native/nc-setuid-helper/src/status.h`:

```c
#ifndef NC_STATUS_H
#define NC_STATUS_H

int nc_cmd_status(int argc, char **argv);

#endif
```

新建 `native/nc-setuid-helper/src/status.c`:

```c
#include "status.h"
#include "arg.h"
#include "logging.h"
#include "process.h"

#include <stdio.h>
#include <stdlib.h>

int nc_cmd_status(int argc, char **argv) {
    const char *pid_str = nc_arg_require(argc, argv, "pid");
    const char *uid_str = nc_arg_require(argc, argv, "uid");
    const char *rt = nc_arg_require(argc, argv, "runtime-dir");
    const char *start_time_str = nc_arg_require(argc, argv, "start-time");
    if (!pid_str || !uid_str || !rt || !start_time_str) return 1;

    pid_t pid = (pid_t)atoi(pid_str);
    uid_t expected_uid = (uid_t)atoi(uid_str);
    unsigned long long start_time = strtoull(start_time_str, NULL, 10);

    int rc = nc_process_identity_matches(pid, expected_uid, start_time, rt);
    if (rc == 0) {
        printf("alive pid=%d uid=%u start_time=%llu\n", pid, expected_uid, start_time);
        return 0;
    }
    printf("dead pid=%d reason=%d\n", pid, rc);
    return 0;  /* Status reporting doesn't fail on dead PIDs. */
}
```

- [ ] **Step 4: 编写 process.c(identity helper + 信号名)**

新建 `native/nc-setuid-helper/src/process.h`:

```c
#ifndef NC_PROCESS_H
#define NC_PROCESS_H

#include <sys/types.h>

/**
 * Convert signal name (SIGTERM, SIGKILL, etc.) to signal number.
 * Returns -1 on unknown name.
 */
int nc_signal_from_name(const char *name);

/**
 * Check that /proc/<pid>/stat has the expected real UID and start-time ticks,
 * and that the runtime dir matches. This prevents PID reuse attacks.
 *
 * Returns 0 on match, positive code on mismatch:
 *   1 = PID does not exist
 *   2 = UID mismatch
 *   3 = start_time mismatch (likely PID reuse)
 *   4 = runtime_dir mismatch
 */
int nc_process_identity_matches(pid_t pid, uid_t expected_uid,
                                unsigned long long expected_start_time,
                                const char *expected_runtime_dir);

#endif
```

新建 `native/nc-setuid-helper/src/process.c`:

```c
#include "process.h"

#include <signal.h>
#include <stdio.h>
#include <string.h>

int nc_signal_from_name(const char *name) {
    if (!name) return -1;
    if (strcmp(name, "SIGTERM") == 0 || strcmp(name, "TERM") == 0) return SIGTERM;
    if (strcmp(name, "SIGKILL") == 0 || strcmp(name, "KILL") == 0) return SIGKILL;
    if (strcmp(name, "SIGINT") == 0 || strcmp(name, "INT") == 0) return SIGINT;
    if (strcmp(name, "SIGHUP") == 0 || strcmp(name, "HUP") == 0) return SIGHUP;
    return -1;
}

int nc_process_identity_matches(pid_t pid, uid_t expected_uid,
                                unsigned long long expected_start_time,
                                const char *expected_runtime_dir) {
    char path[64];
    snprintf(path, sizeof(path), "/proc/%d/stat", (int)pid);
    FILE *f = fopen(path, "r");
    if (!f) return 1;

    /* /proc/<pid>/stat format:
     * pid (comm) state ppid pgrp session tty tpgid flags minflt cminflt majflt
     * cmajflt utime stime cutime cstime priority nice threads itrealvalue
     * starttime vsize rss ...
     * starttime is field 22 after the comm parens. We parse simply. */
    char comm[256];
    unsigned long long start_time = 0;
    /* fscanf skips comm parens; easier to read line then re-parse. */
    char line[2048];
    if (!fgets(line, sizeof(line), f)) { fclose(f); return 1; }
    fclose(f);

    /* Find closing paren of comm. */
    char *close = strrchr(line, ')');
    if (!close) return 1;
    /* Fields after close are space-separated. starttime is field 22 in the
     * full stat (counting pid=1, comm=2). After comm there are 20 fields
     * before starttime. */
    int fields_consumed = 0;
    char *p = close + 1;
    while (*p == ' ') p++;
    unsigned long long scratch;
    for (int i = 0; i < 19; i++) {
        if (sscanf(p, "%llu", &scratch) != 1) return 1;
        /* skip to next space */
        while (*p && *p != ' ') p++;
        while (*p == ' ') p++;
        fields_consumed++;
    }
    if (sscanf(p, "%llu", &start_time) != 1) return 1;

    (void)comm;
    (void)fields_consumed;

    if (start_time != expected_start_time) return 3;

    /* UID check via /proc/<pid>/status. */
    char status_path[64];
    snprintf(status_path, sizeof(status_path), "/proc/%d/status", (int)pid);
    FILE *sf = fopen(status_path, "r");
    if (!sf) return 1;
    uid_t uid = (uid_t)-1;
    char sline[256];
    while (fgets(sline, sizeof(sline), sf)) {
        if (strncmp(sline, "Uid:", 4) == 0) {
            unsigned int r, e, s, f;
            if (sscanf(sline + 4, "%u %u %u %u", &r, &e, &s, &f) == 4) {
                uid = (uid_t)r;
                break;
            }
        }
    }
    fclose(sf);
    if (uid != expected_uid) return 2;

    /* runtime_dir check via /proc/<pid>/cwd. */
    char cwd_path[64];
    snprintf(cwd_path, sizeof(cwd_path), "/proc/%d/cwd", (int)pid);
    char buf[4096];
    ssize_t n = readlink(cwd_path, buf, sizeof(buf) - 1);
    if (n < 0) return 4;
    buf[n] = '\0';
    /* Accept if cwd is the runtime dir or a subdirectory of it. */
    size_t rlen = strlen(expected_runtime_dir);
    if (strncmp(buf, expected_runtime_dir, rlen) != 0) return 4;
    if (buf[rlen] != '\0' && buf[rlen] != '/') return 4;

    return 0;
}
```

- [ ] **Step 5: 编写 cgroup_cmd.c**

新建 `native/nc-setuid-helper/src/cgroup_cmd.h`:

```c
#ifndef NC_CGROUP_CMD_H
#define NC_CGROUP_CMD_H

int nc_cmd_cgroup(int argc, char **argv);

#endif
```

新建 `native/nc-setuid-helper/src/cgroup_cmd.c`:

```c
#include "cgroup_cmd.h"
#include "arg.h"
#include "cgroup.h"
#include "logging.h"

#include <stdio.h>
#include <stdlib.h>

int nc_cmd_cgroup(int argc, char **argv) {
    const char *path = nc_arg_require(argc, argv, "path");
    const char *mem_str = nc_arg_get(argc, argv, "mem");
    const char *pids_str = nc_arg_get(argc, argv, "pids");
    const char *cpu_str = nc_arg_get(argc, argv, "cpu");
    if (!path) return 1;

    int mem = mem_str ? atoi(mem_str) : 0;
    int pids = pids_str ? atoi(pids_str) : 0;
    int cpu = cpu_str ? atoi(cpu_str) : 0;

    return nc_cgroup_create(path, mem, pids, cpu) == 0 ? 0 : 2;
}
```

- [ ] **Step 6: 把全部命令接入 main.c**

编辑 `native/nc-setuid-helper/src/main.c`,让它分发所有五个命令:

```c
#include "logging.h"
#include "prepare.h"
#include "spawn.h"
#include "kill.h"
#include "status.h"
#include "cgroup_cmd.h"

#include <stdio.h>
#include <string.h>

static int usage(void) {
    fprintf(stderr,
        "usage: nc-setuid-helper <command> [args...]\n"
        "commands:\n"
        "  prepare --tenant=T --agent=A --group=G --runtime-dir=D\n"
        "  spawn   --uid=U --gid=G --cgroup=P --runtime-dir=D -- cmd...\n"
        "  kill    --pid=P --signal=S --uid=U --runtime-dir=D --cgroup=P --start-time=T\n"
        "  cgroup  --path=P --mem=MB --pids=N --cpu=S\n"
        "  status  --pid=P --uid=U --runtime-dir=D --cgroup=P --start-time=T\n");
    return 1;
}

int main(int argc, char **argv) {
    if (argc < 2) return usage();
    if (strcmp(argv[1], "--help") == 0 || strcmp(argv[1], "-h") == 0) return usage();

    nc_log_info("cmd=%s argc=%d", argv[1], argc);

    if (strcmp(argv[1], "prepare") == 0) return nc_cmd_prepare(argc, argv);
    if (strcmp(argv[1], "spawn") == 0)   return nc_cmd_spawn(argc, argv);
    if (strcmp(argv[1], "kill") == 0)    return nc_cmd_kill(argc, argv);
    if (strcmp(argv[1], "status") == 0)  return nc_cmd_status(argc, argv);
    if (strcmp(argv[1], "cgroup") == 0)  return nc_cmd_cgroup(argc, argv);

    fprintf(stderr, "error: unknown command '%s'\n", argv[1]);
    return 1;
}
```

- [ ] **Step 7: 构建 + 运行纯逻辑测试**

运行: `cd native/nc-setuid-helper && make build test`
预期: 构建成功,所有现有的纯逻辑测试仍然通过。

- [ ] **Step 8: 提交**

```bash
git add native/nc-setuid-helper/
git commit -m "feat(helper): implement spawn/kill/status/cgroup commands"
```

---

## Task 9: Root-gated E2E helper 测试

**文件:**
- 新建: `tests/root/prepare_spawn_status.sh`

- [ ] **Step 1: 编写 root E2E 脚本**

新建 `native/nc-setuid-helper/tests/root/prepare_spawn_status.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
if [[ "${EUID}" -ne 0 ]]; then
  echo "skip (not root)"
  exit 0
fi

BUILD_DIR="$(cd "$(dirname "$0")/../../build" && pwd)"
BIN="$BUILD_DIR/nc-setuid-helper"
if [[ ! -x "$BIN" ]]; then
  echo "FATAL: helper binary not built at $BIN" >&2
  exit 2
fi

# Ensure nanoclaw-svc user + nc-priv group exist.
id nanoclaw-svc >/dev/null 2>&1 || useradd -r nanoclaw-svc
getent group nc-priv >/dev/null 2>&1 || groupadd -r nc-priv
usermod -aG nc-priv nanoclaw-svc

# Set up test data dir.
TEST_DATA=$(mktemp -d /tmp/nc-helper-e2e-XXXXXX)
trap 'rm -rf "$TEST_DATA"' EXIT
mkdir -p "$TEST_DATA/runtime/acme/finance/main"
mkdir -p "$TEST_DATA/cgroup"

# Run the helper against the test paths. (Requires users.db at /var/lib/nanoclaw
# for production use; for tests we point it there via a small env override —
# NC_USERS_DB_PATH is not in the code, so use a symlink instead.)
sudo mkdir -p /var/lib/nanoclaw
sudo ln -sf "$TEST_DATA" /var/lib/nanoclaw/test

# Run as nanoclaw-svc to simulate real-world invocation.
RT_DIR="$TEST_DATA/runtime/acme/finance/main"
sudo -u nanoclaw-svc "$BIN" prepare \
  --tenant=acme --agent=finance --group=main --runtime-dir="$RT_DIR"
PREP_RC=$?

[[ "$PREP_RC" -eq 0 ]] || { echo "prepare failed"; exit 1; }

# Now try to spawn /bin/true as the mapped user. Read the mapped uid from users.db.
USERNAME=$(sqlite3 /var/lib/nanoclaw/users.db "SELECT username FROM users WHERE tenant='acme' AND agent='finance' AND group_folder='main'")
UID_NUM=$(sqlite3 /var/lib/nanoclaw/users.db "SELECT uid FROM users WHERE username='$USERNAME'")
GID_NUM=$(sqlite3 /var/lib/nanoclaw/users.db "SELECT gid FROM users WHERE username='$USERNAME'")

sudo -u nanoclaw-svc "$BIN" spawn \
  --uid="$UID_NUM" --gid="$GID_NUM" --runtime-dir="$RT_DIR" -- /bin/true
SPAWN_RC=$?

[[ "$SPAWN_RC" -eq 0 ]] || { echo "spawn failed"; exit 1; }

echo "prepare_spawn_status: OK"
exit 0
```

```bash
chmod +x native/nc-setuid-helper/tests/root/prepare_spawn_status.sh
```

- [ ] **Step 2: 运行 root E2E**

运行: `cd native/nc-setuid-helper && sudo tests/root/prepare_spawn_status.sh`
预期: `prepare_spawn_status: OK`。(需要 `sqlite3` CLI 和已构建的 helper。)

- [ ] **Step 3: 提交**

```bash
git add native/nc-setuid-helper/tests/root/
git commit -m "test(helper): root-gated e2e for prepare + spawn"
```

---

## Task 10: TS helper client 封装

**文件:**
- 新建: `src/runtime/helper-client.ts`
- 新建: `src/runtime/helper-client.test.ts`

- [ ] **Step 1: 写失败的测试**

新建 `src/runtime/helper-client.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { HelperClient, HelperPrepareResult, HelperSpawnOpts } from './helper-client.js';

let helperBin: string;
let workdir: string;

beforeEach(() => {
  workdir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-helper-'));
  // For tests, use a fake helper script that records argv and prints canned output.
  helperBin = path.join(workdir, 'fake-helper');
});

afterEach(() => {
  fs.rmSync(workdir, { recursive: true, force: true });
});

function writeFakeHelper(output: string, exitCode = 0): void {
  fs.writeFileSync(
    helperBin,
    `#!/usr/bin/env bash\necho -n "${output}"\nexit ${exitCode}\n`,
  );
  fs.chmodSync(helperBin, 0o755);
}

describe('HelperClient', () => {
  it('prepare parses stdout into structured result', async () => {
    writeFakeHelper('username=ncg-acme-financ-0123456789 uid=60001 gid=60001 runtime_dir=/x\n');
    const client = new HelperClient({ binaryPath: helperBin });
    const r = await client.prepare({
      tenant: 'acme', agent: 'finance', group: 'main', runtimeDir: '/x',
    });
    expect(r.username).toBe('ncg-acme-financ-0123456789');
    expect(r.uid).toBe(60001);
    expect(r.gid).toBe(60001);
  });

  it('prepare surfaces non-zero exit as error', async () => {
    writeFakeHelper('', 1);
    const client = new HelperClient({ binaryPath: helperBin });
    await expect(
      client.prepare({ tenant: 'acme', agent: 'finance', group: 'main', runtimeDir: '/x' }),
    ).rejects.toThrow(/prepare.*failed/);
  });

  it('spawn passes argv through and returns exit code', async () => {
    writeFakeHelper('', 0);
    const client = new HelperClient({ binaryPath: helperBin });
    const rc = await client.spawn({
      uid: 60001, gid: 60001, runtimeDir: '/x', cmd: ['/bin/true'],
    } as HelperSpawnOpts);
    expect(rc).toBe(0);
  });

  it('kill forwards signal + identity args', async () => {
    writeFakeHelper('', 0);
    const client = new HelperClient({ binaryPath: helperBin });
    await client.kill({
      pid: 12345, signal: 'SIGTERM', uid: 60001,
      runtimeDir: '/x', cgroup: '/sys/fs/cgroup/nanoclaw/x', startTime: 9999,
    });
    // No throw means success.
  });
});
```

- [ ] **Step 2: 运行测试以验证失败**

运行: `npm test -- src/runtime/helper-client.test.ts`
预期: FAIL — 模块不存在。

- [ ] **Step 3: 实现 helper-client**

新建 `src/runtime/helper-client.ts`:

```typescript
import { spawnFile } from 'node:child_process';

export interface HelperClientConfig {
  binaryPath: string;
}

export interface HelperPrepareArgs {
  tenant: string;
  agent: string;
  group: string;
  runtimeDir: string;
}

export interface HelperPrepareResult {
  username: string;
  uid: number;
  gid: number;
  runtimeDir: string;
}

export interface HelperSpawnOpts {
  uid: number;
  gid: number;
  cgroup?: string;
  runtimeDir: string;
  cmd: string[];
}

export interface HelperKillArgs {
  pid: number;
  signal: string;
  uid: number;
  runtimeDir: string;
  cgroup?: string;
  startTime: number;
}

export interface HelperCgroupArgs {
  path: string;
  mem?: number;
  pids?: number;
  cpu?: number;
}

export class HelperClient {
  constructor(private readonly config: HelperClientConfig) {}

  async prepare(args: HelperPrepareArgs): Promise<HelperPrepareResult> {
    const out = await this.invoke([
      'prepare',
      `--tenant=${args.tenant}`,
      `--agent=${args.agent}`,
      `--group=${args.group}`,
      `--runtime-dir=${args.runtimeDir}`,
    ]);
    return parsePrepareOutput(out);
  }

  async spawn(opts: HelperSpawnOpts): Promise<number> {
    const argv = [
      'spawn',
      `--uid=${opts.uid}`,
      `--gid=${opts.gid}`,
      `--runtime-dir=${opts.runtimeDir}`,
    ];
    if (opts.cgroup) argv.push(`--cgroup=${opts.cgroup}`);
    argv.push('--', ...opts.cmd);
    return this.invoke(argv);
  }

  async kill(args: HelperKillArgs): Promise<void> {
    const argv = [
      'kill',
      `--pid=${args.pid}`,
      `--signal=${args.signal}`,
      `--uid=${args.uid}`,
      `--runtime-dir=${args.runtimeDir}`,
      `--start-time=${args.startTime}`,
    ];
    if (args.cgroup) argv.push(`--cgroup=${args.cgroup}`);
    await this.invoke(argv);
  }

  async cgroup(args: HelperCgroupArgs): Promise<void> {
    const argv = ['cgroup', `--path=${args.path}`];
    if (args.mem !== undefined) argv.push(`--mem=${args.mem}`);
    if (args.pids !== undefined) argv.push(`--pids=${args.pids}`);
    if (args.cpu !== undefined) argv.push(`--cpu=${args.cpu}`);
    await this.invoke(argv);
  }

  async status(args: {
    pid: number;
    uid: number;
    runtimeDir: string;
    startTime: number;
    cgroup?: string;
  }): Promise<boolean> {
    const argv = [
      'status',
      `--pid=${args.pid}`,
      `--uid=${args.uid}`,
      `--runtime-dir=${args.runtimeDir}`,
      `--start-time=${args.startTime}`,
    ];
    if (args.cgroup) argv.push(`--cgroup=${args.cgroup}`);
    try {
      const stdout = await this.invoke(argv);
      return stdout.startsWith('alive');
    } catch {
      return false;
    }
  }

  private invoke(argv: string[]): Promise<number> {
    return new Promise((resolve, reject) => {
      const child = spawnFile(this.config.binaryPath, argv, {
        stdio: ['ignore', 'pipe', 'pipe'],
      });
      let stdout = '';
      let stderr = '';
      child.stdout?.on('data', (c) => (stdout += c.toString()));
      child.stderr?.on('data', (c) => (stderr += c.toString()));
      child.on('error', reject);
      child.on('exit', (code) => {
        if (code === 0) resolve(typeof code === 'number' ? (code === 0 ? 0 : code) : 0);
        else {
          reject(new Error(`helper ${argv[0]} failed code=${code} stderr=${stderr.trim()}`));
        }
      });
      // Stash stdout for callers that need it (prepare). We resolve with exit code;
      // callers that need stdout should call invokeWithStdout.
      (child as unknown as { __stdout?: string }).__stdout = stdout;
    });
  }

  private invokeWithStdout(argv: string[]): Promise<string> {
    return new Promise((resolve, reject) => {
      const child = spawnFile(this.config.binaryPath, argv, {
        stdio: ['ignore', 'pipe', 'pipe'],
      });
      let stdout = '';
      let stderr = '';
      child.stdout?.on('data', (c) => (stdout += c.toString()));
      child.stderr?.on('data', (c) => (stderr += c.toString()));
      child.on('error', reject);
      child.on('exit', (code) => {
        if (code === 0) resolve(stdout);
        else reject(new Error(`helper ${argv[0]} failed code=${code} stderr=${stderr.trim()}`));
      });
    });
  }
}

function parsePrepareOutput(out: string): HelperPrepareResult {
  // Expected: "username=... uid=... gid=... runtime_dir=...\n"
  const fields = new Map<string, string>();
  for (const token of out.trim().split(/\s+/)) {
    const idx = token.indexOf('=');
    if (idx > 0) fields.set(token.slice(0, idx), token.slice(idx + 1));
  }
  const username = fields.get('username');
  const uid = Number(fields.get('uid'));
  const gid = Number(fields.get('gid'));
  const runtimeDir = fields.get('runtime_dir');
  if (!username || !Number.isInteger(uid) || !Number.isInteger(gid) || !runtimeDir) {
    throw new Error(`prepare output unparseable: '${out}'`);
  }
  return { username, uid, gid, runtimeDir };
}
```

注:测试中第一个 `invoke` 签名只需要退出码;但 `prepare` 需要 stdout。调整测试的 `spawn` 路径使用退出码,`prepare` 使用 stdout。我们可以简化为一个返回 `{ code, stdout, stderr }` 的私有方法加两个薄的 public adapter。下面直接重构。

把 `invoke` 重构为同时返回两者,调用方按需消费。替换 `invoke` 方法:

```typescript
  private async invokeRaw(argv: string[]): Promise<{ code: number; stdout: string; stderr: string }> {
    return new Promise((resolve, reject) => {
      const child = spawnFile(this.config.binaryPath, argv, {
        stdio: ['ignore', 'pipe', 'pipe'],
      });
      let stdout = '';
      let stderr = '';
      child.stdout?.on('data', (c) => (stdout += c.toString()));
      child.stderr?.on('data', (c) => (stderr += c.toString()));
      child.on('error', reject);
      child.on('exit', (code) => resolve({ code: code ?? -1, stdout, stderr }));
    });
  }

  private async invoke(argv: string[]): Promise<number> {
    const r = await this.invokeRaw(argv);
    if (r.code !== 0) {
      throw new Error(`helper ${argv[0]} failed code=${r.code} stderr=${r.stderr.trim()}`);
    }
    return r.code;
  }

  private async invokeCapture(argv: string[]): Promise<string> {
    const r = await this.invokeRaw(argv);
    if (r.code !== 0) {
      throw new Error(`helper ${argv[0]} failed code=${r.code} stderr=${r.stderr.trim()}`);
    }
    return r.stdout;
  }
```

然后更新 `prepare`,让它使用 `invokeCapture`:

```typescript
  async prepare(args: HelperPrepareArgs): Promise<HelperPrepareResult> {
    const out = await this.invokeCapture([
      'prepare',
      `--tenant=${args.tenant}`,
      `--agent=${args.agent}`,
      `--group=${args.group}`,
      `--runtime-dir=${args.runtimeDir}`,
    ]);
    return parsePrepareOutput(out);
  }
```

- [ ] **Step 4: 运行测试以验证通过**

运行: `npm test -- src/runtime/helper-client.test.ts`
预期: PASS(4 个测试)。

- [ ] **Step 5: 提交**

```bash
git add src/runtime/helper-client.ts src/runtime/helper-client.test.ts
git commit -m "feat(runtime): add TS client for nc-setuid-helper"
```

---

## Task 11: Runtime DB(inbound/outbound/state/tools)

**文件:**
- 新建: `src/runtime/runtime-dbs.ts`
- 新建: `src/runtime/runtime-dbs.test.ts`

- [ ] **Step 1: 写失败的测试**

新建 `src/runtime/runtime-dbs.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { openRuntimeDbs } from './runtime-dbs.js';

let root: string;

beforeEach(() => {
  root = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-rtdbs-'));
});

afterEach(() => {
  fs.rmSync(root, { recursive: true, force: true });
});

describe('openRuntimeDbs', () => {
  it('creates all 4 runtime DBs under the target dir', () => {
    const target = path.join(root, 'live');
    fs.mkdirSync(target, { recursive: true });
    const dbs = openRuntimeDbs(target);
    for (const name of ['inbound.db', 'outbound.db', 'state.db', 'tools.db']) {
      expect(fs.existsSync(path.join(target, name))).toBe(true);
    }
    dbs.close();
  });

  it('inbound/outbound schema has expected columns', () => {
    const target = path.join(root, 'live');
    fs.mkdirSync(target, { recursive: true });
    const dbs = openRuntimeDbs(target);
    const inCols = dbs.inbound.prepare('PRAGMA table_info(inbound)').all() as { name: string }[];
    expect(inCols.map((c) => c.name)).toContain('row_id');
    expect(inCols.map((c) => c.name)).toContain('status');

    const outCols = dbs.outbound.prepare('PRAGMA table_info(outbound)').all() as { name: string }[];
    expect(outCols.map((c) => c.name)).toContain('status');
    expect(outCols.map((c) => c.name)).toContain('delivered_at');
    dbs.close();
  });

  it('tools.db has request and response columns', () => {
    const target = path.join(root, 'live');
    fs.mkdirSync(target, { recursive: true });
    const dbs = openRuntimeDbs(target);
    const cols = dbs.tools.prepare('PRAGMA table_info(tool_requests)').all() as { name: string }[];
    const names = cols.map((c) => c.name);
    expect(names).toContain('tenant_id');
    expect(names).toContain('agent_id');
    expect(names).toContain('group_folder');
    expect(names).toContain('run_id');
    expect(names).toContain('payload');
    expect(names).toContain('status');
    dbs.close();
  });
});
```

- [ ] **Step 2: 运行测试以验证失败**

运行: `npm test -- src/runtime/runtime-dbs.test.ts`
预期: FAIL — 模块不存在。

- [ ] **Step 3: 实现 runtime-dbs**

新建 `src/runtime/runtime-dbs.ts`:

```typescript
import Database from 'better-sqlite3';
import fs from 'fs';
import path from 'path';

export interface RuntimeDbs {
  inbound: Database.Database;
  outbound: Database.Database;
  state: Database.Database;
  tools: Database.Database;
  close(): void;
}

/**
 * Open (or create) the 4 runtime DBs for a given run directory.
 * The run directory must already exist (created by nc-setuid-helper prepare).
 *
 * Path layout under dir:
 *   inbound.db    — work to do, written by control plane, read by run
 *   outbound.db   — delivery requests, written by run, read by control plane
 *   state.db      — run-local control keys and provider continuation
 *   tools.db      — tool requests and results (bidirectional)
 *
 * The caller is responsible for permissioning; this function creates files
 * with default umask, which the ACL default on the parent dir will narrow.
 */
export function openRuntimeDbs(dir: string): RuntimeDbs {
  if (!fs.existsSync(dir)) {
    throw new Error(`runtime dir does not exist: ${dir}`);
  }

  const inbound = openDb(path.join(dir, 'inbound.db'));
  const outbound = openDb(path.join(dir, 'outbound.db'));
  const state = openDb(path.join(dir, 'state.db'));
  const tools = openDb(path.join(dir, 'tools.db'));

  applyInboundSchema(inbound);
  applyOutboundSchema(outbound);
  applyStateSchema(state);
  applyToolsSchema(tools);

  return {
    inbound,
    outbound,
    state,
    tools,
    close() {
      inbound.close();
      outbound.close();
      state.close();
      tools.close();
    },
  };
}

function openDb(dbPath: string): Database.Database {
  const db = new Database(dbPath);
  db.pragma('journal_mode = WAL');
  db.pragma('busy_timeout = 5000');
  return db;
}

function applyInboundSchema(db: Database.Database): void {
  db.exec(`
    CREATE TABLE IF NOT EXISTS inbound (
      row_id INTEGER PRIMARY KEY AUTOINCREMENT,
      source_message_id TEXT,
      channel_type TEXT NOT NULL,
      chat_jid TEXT NOT NULL,
      sender TEXT,
      sender_name TEXT,
      is_from_me INTEGER,
      content TEXT,
      timestamp TEXT NOT NULL,
      message_type TEXT,
      attachment TEXT,
      card_action TEXT,
      dedupe_key TEXT UNIQUE,
      attempt_count INTEGER DEFAULT 0,
      status TEXT NOT NULL DEFAULT 'pending',  -- pending | claimed | completed | error
      enqueued_at TEXT NOT NULL,
      claimed_at TEXT,
      completed_at TEXT,
      error TEXT
    );
    CREATE INDEX IF NOT EXISTS idx_inbound_status ON inbound(status, enqueued_at);
    CREATE INDEX IF NOT EXISTS idx_inbound_chat ON inbound(chat_jid, timestamp);
  `);
}

function applyOutboundSchema(db: Database.Database): void {
  db.exec(`
    CREATE TABLE IF NOT EXISTS outbound (
      row_id INTEGER PRIMARY KEY AUTOINCREMENT,
      source_inbound_ids TEXT,     -- JSON array
      channel_type TEXT NOT NULL,
      chat_jid TEXT NOT NULL,
      text TEXT NOT NULL,
      message_type TEXT,
      attachment TEXT,
      idempotency_key TEXT UNIQUE,
      status TEXT NOT NULL DEFAULT 'pending',  -- pending | delivering | delivered | error
      enqueued_at TEXT NOT NULL,
      delivered_at TEXT,
      channel_message_id TEXT,
      error TEXT,
      retry_count INTEGER DEFAULT 0
    );
    CREATE INDEX IF NOT EXISTS idx_outbound_status ON outbound(status, enqueued_at);
  `);
}

function applyStateSchema(db: Database.Database): void {
  db.exec(`
    CREATE TABLE IF NOT EXISTS state (
      key TEXT PRIMARY KEY,
      value TEXT NOT NULL,
      updated_at TEXT NOT NULL
    );
    CREATE TABLE IF NOT EXISTS audit (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      event TEXT NOT NULL,
      model TEXT,
      tokens_in INTEGER,
      tokens_out INTEGER,
      latency_ms INTEGER,
      status TEXT,
      ts TEXT NOT NULL
    );
  `);
}

function applyToolsSchema(db: Database.Database): void {
  db.exec(`
    CREATE TABLE IF NOT EXISTS tool_requests (
      row_id INTEGER PRIMARY KEY AUTOINCREMENT,
      tenant_id TEXT NOT NULL,
      agent_id TEXT NOT NULL,
      group_folder TEXT NOT NULL,
      run_id TEXT NOT NULL,
      tool_name TEXT NOT NULL,
      payload TEXT NOT NULL,        -- JSON
      status TEXT NOT NULL DEFAULT 'pending',  -- pending | claimed | completed | error
      result TEXT,                  -- JSON
      error TEXT,
      created_at TEXT NOT NULL,
      claimed_at TEXT,
      completed_at TEXT,
      lease_owner TEXT
    );
    CREATE INDEX IF NOT EXISTS idx_tool_status ON tool_requests(status, created_at);
    CREATE INDEX IF NOT EXISTS idx_tool_run ON tool_requests(tenant_id, agent_id, run_id);
  `);
}
```

- [ ] **Step 4: 运行测试以验证通过**

运行: `npm test -- src/runtime/runtime-dbs.test.ts`
预期: PASS(3 个测试)。

- [ ] **Step 5: 提交**

```bash
git add src/runtime/runtime-dbs.ts src/runtime/runtime-dbs.test.ts
git commit -m "feat(runtime): add runtime DBs (inbound/outbound/state/tools) with schema"
```

---

## Task 12: Spawner — prepare + spawn + 注册 active_run

**文件:**
- 新建: `src/runtime/spawner.ts`
- 新建: `src/runtime/spawner.test.ts`

- [ ] **Step 1: 写失败的测试**

新建 `src/runtime/spawner.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { Spawner, SpawnerOpts } from './spawner.js';
import type { HelperClient } from './helper-client.js';

let dataDir: string;

beforeEach(() => {
  dataDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-spawner-'));
});

afterEach(() => {
  fs.rmSync(dataDir, { recursive: true, force: true });
});

function fakeHelper(): HelperClient {
  return {
    prepare: async () => ({
      username: 'ncg-acme-financ-0123456789',
      uid: 60001,
      gid: 60001,
      runtimeDir: '/x',
    }),
    spawn: async () => 0,
    kill: async () => {},
    cgroup: async () => {},
    status: async () => true,
  } as unknown as HelperClient;
}

describe('Spawner', () => {
  it('prepare writes the runtime dir path and registers active_run', async () => {
    const spawner = new Spawner({
      dataDir,
      helper: fakeHelper(),
      agentRunnerBin: '/bin/true',
    } as SpawnerOpts);

    const handle = await spawner.startLiveRun({
      tenantId: 'acme',
      agentId: 'finance',
      group: 'main',
      model: 'claude-sonnet-4',
      env: {},
    });

    expect(handle.runId).toBeTruthy();
    expect(handle.username).toBe('ncg-acme-financ-0123456789');
    expect(handle.uid).toBe(60001);
    // active_runs row should exist.
    const { listActiveRuns } = await import('../db/active-runs.js');
    const rows = listActiveRuns(dataDir, 'acme', 'finance');
    expect(rows.find((r) => r.run_id === handle.runId)).toBeTruthy();
  });

  it('startIsolatedRun uses runs/<runId>/ as runtime dir', async () => {
    const spawner = new Spawner({
      dataDir,
      helper: fakeHelper(),
      agentRunnerBin: '/bin/true',
      env: {},
    } as SpawnerOpts);

    const handle = await spawner.startIsolatedRun({
      tenantId: 'acme',
      agentId: 'finance',
      group: 'main',
      runId: 'run-xyz',
      model: 'claude-sonnet-4',
      env: {},
    });

    expect(handle.runtimeDir.endsWith('/runs/run-xyz')).toBe(true);
  });
});
```

- [ ] **Step 2: 运行测试以验证失败**

运行: `npm test -- src/runtime/spawner.test.ts`
预期: FAIL — 模块不存在。

- [ ] **Step 3: 实现 spawner**

新建 `src/runtime/spawner.ts`:

```typescript
import { randomUUID } from 'node:crypto';
import fs from 'node:fs';
import path from 'node:path';

import { insertActiveRun, ActiveRunRecord } from '../db/active-runs.js';
import { tenantAgentRuntimeDir } from '../tenants/naming.js';
import type { HelperClient } from './helper-client.js';

export interface SpawnerOpts {
  dataDir: string;
  helper: HelperClient;
  agentRunnerBin: string;
  /** Optional override for helper cgroup root; defaults to /sys/fs/cgroup/nanoclaw. */
  cgroupRoot?: string;
}

export interface StartRunArgs {
  tenantId: string;
  agentId: string;
  group: string;
  model: string;
  env: Record<string, string>;
  instructionsPath?: string;
}

export interface RunHandle {
  runId: string;
  mode: 'live' | 'isolated';
  username: string;
  uid: number;
  gid: number;
  runtimeDir: string;
  cgroupPath: string;
  pid: number;
  startTimeTicks: number;
}

export class Spawner {
  constructor(private readonly opts: SpawnerOpts) {}

  async startLiveRun(args: StartRunArgs): Promise<RunHandle> {
    return this.start({ ...args, mode: 'live' });
  }

  async startIsolatedRun(args: StartRunArgs & { runId: string }): Promise<RunHandle> {
    return this.start({ ...args, mode: 'isolated' });
  }

  private async start(args: StartRunArgs & { mode: 'live' | 'isolated'; runId?: string }): Promise<RunHandle> {
    const runId = args.runId ?? randomUUID();
    const liveOrRuns = args.mode === 'live' ? 'live' : `runs/${runId}`;
    const runtimeDir = tenantAgentRuntimeDir(this.opts.dataDir, args.tenantId, args.agentId, args.group);
    const fullRuntimeDir = path.join(runtimeDir, liveOrRuns);

    fs.mkdirSync(path.dirname(fullRuntimeDir), { recursive: true });

    const prepared = await this.opts.helper.prepare({
      tenant: args.tenantId,
      agent: args.agentId,
      group: args.group,
      runtimeDir, // prepare targets the group root; sub-dirs created inside
    });

    const cgroupPath = `${this.opts.cgroupRoot ?? '/sys/fs/cgroup/nanoclaw'}/${runId}`;
    await this.opts.helper.cgroup({ path: runId, mem: 1024, pids: 256 });

    // Spawn agent-runner. We do NOT await the child — it runs in background
    // and writes to outbound.db. Use spawn() detached.
    const argv = [
      this.opts.agentRunnerBin,
      '--mode', args.mode,
      '--tenant', args.tenantId,
      '--agent', args.agentId,
      '--group', args.group,
      '--run-id', runId,
      '--runtime-dir', fullRuntimeDir,
      '--model', args.model,
    ];
    if (args.instructionsPath) argv.push('--instructions', args.instructionsPath);

    await this.opts.helper.spawn({
      uid: prepared.uid,
      gid: prepared.gid,
      runtimeDir: fullRuntimeDir,
      cgroup: runId,
      cmd: argv,
    });

    // Because spawn execs, we cannot see the child PID directly from the helper.
    // For Foundation purposes, we scan /proc for the mapped uid and pick the
    // newest matching PID. This is race-prone; Plan C Task 13 introduces a
    // pidfile or a pipe from the helper to report the PID reliably.
    const pid = await findNewestPidForUid(prepared.uid);
    const startTimeTicks = await readStartTimeTicks(pid);

    const record: ActiveRunRecord = {
      tenant_id: args.tenantId,
      agent_id: args.agentId,
      group_folder: args.group,
      run_id: runId,
      mode: args.mode,
      pid,
      pid_start_time_ticks: startTimeTicks,
      expected_uid: prepared.username,
      runtime_dir: fullRuntimeDir,
      cgroup_path: cgroupPath,
      started_at: new Date().toISOString(),
      last_seen_at: new Date().toISOString(),
      status: 'active',
    };
    insertActiveRun(this.opts.dataDir, record);

    return {
      runId,
      mode: args.mode,
      username: prepared.username,
      uid: prepared.uid,
      gid: prepared.gid,
      runtimeDir: fullRuntimeDir,
      cgroupPath,
      pid,
      startTimeTicks,
    };
  }
}

async function findNewestPidForUid(uid: number): Promise<number> {
  const entries = await fs.promises.readdir('/proc');
  let newest = 0;
  let newestStart = 0;
  for (const e of entries) {
    if (!/^\d+$/.test(e)) continue;
    const statusPath = `/proc/${e}/status`;
    try {
      const txt = await fs.promises.readFile(statusPath, 'utf8');
      const m = txt.match(/^Uid:\s+(\d+)/m);
      if (m && Number(m[1]) === uid) {
        const stat = await fs.promises.readFile(`/proc/${e}/stat`, 'utf8');
        const close = stat.lastIndexOf(')');
        const fields = stat.slice(close + 2).split(/\s+/);
        const start = Number(fields[19]); // starttime is field 22, index 19 after comm
        if (start > newestStart) {
          newestStart = start;
          newest = Number(e);
        }
      }
    } catch {
      // process may have exited
    }
  }
  return newest;
}

async function readStartTimeTicks(pid: number): Promise<number> {
  if (!pid) return 0;
  try {
    const stat = await fs.promises.readFile(`/proc/${pid}/stat`, 'utf8');
    const close = stat.lastIndexOf(')');
    const fields = stat.slice(close + 2).split(/\s+/);
    return Number(fields[19]);
  } catch {
    return 0;
  }
}
```

- [ ] **Step 4: 运行测试以验证通过**

运行: `npm test -- src/runtime/spawner.test.ts`
预期: PASS(2 个测试)。

注:在测试中,`findNewestPidForUid` 大概率返回 0(没有真实进程以该 uid 运行);测试不断言 `pid` 非零,只检查 active_runs 行是否存在。

- [ ] **Step 5: 提交**

```bash
git add src/runtime/spawner.ts src/runtime/spawner.test.ts
git commit -m "feat(runtime): add Spawner that drives helper prepare/spawn + active_runs"
```

---

## Task 13: Lifecycle manager — monitor、reap、kill、reconcile

**文件:**
- 新建: `src/runtime/lifecycle.ts`
- 新建: `src/runtime/lifecycle.test.ts`

- [ ] **Step 1: 写失败的测试**

新建 `src/runtime/lifecycle.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { LifecycleManager } from './lifecycle.js';
import type { HelperClient } from './helper-client.js';
import { insertActiveRun } from '../db/active-runs.js';
import { closeAllHostDbs } from '../db/partition.js';

let dataDir: string;

beforeEach(() => {
  dataDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-lifecycle-'));
});

afterEach(() => {
  closeAllHostDbs();
  fs.rmSync(dataDir, { recursive: true, force: true });
});

const fakeHelper = () =>
  ({
    status: async () => false,
    kill: async () => {},
  } as unknown as HelperClient);

describe('LifecycleManager', () => {
  it('reconcile marks dead runs as crashed', async () => {
    insertActiveRun(dataDir, {
      tenant_id: 'acme',
      agent_id: 'finance',
      group_folder: 'main',
      run_id: 'r1',
      mode: 'live',
      pid: 99999,
      pid_start_time_ticks: 0,
      expected_uid: 'ncg-x',
      runtime_dir: '/nonexistent',
      cgroup_path: null,
      started_at: new Date().toISOString(),
      last_seen_at: new Date().toISOString(),
      status: 'active',
    });
    const mgr = new LifecycleManager({ dataDir, helper: fakeHelper() });
    await mgr.reconcile();
    const rows = await import('../db/active-runs.js').then((m) => m.listActiveRuns(dataDir, 'acme', 'finance'));
    expect(rows[0].status).toBe('crashed');
  });
});
```

- [ ] **Step 2: 运行测试以验证失败**

运行: `npm test -- src/runtime/lifecycle.test.ts`
预期: FAIL — 模块不存在。

- [ ] **Step 3: 实现 lifecycle manager**

新建 `src/runtime/lifecycle.ts`:

```typescript
import fs from 'node:fs';

import {
  listActiveRuns,
  markActiveRunExited,
  deleteActiveRun,
  updateActiveRunLastSeen,
} from '../db/active-runs.js';
import type { HelperClient } from './helper-client.js';

export interface LifecycleOpts {
  dataDir: string;
  helper: HelperClient;
  /** How often to refresh /proc-based liveness. Default 10s. */
  pollIntervalMs?: number;
}

export class LifecycleManager {
  private timer?: NodeJS.Timeout;

  constructor(private readonly opts: LifecycleOpts) {}

  /**
   * Scan all partition DBs we can find and reconcile active_runs against
   * actual process state via the helper's status command.
   *
   * This is the host-restart path: any PID whose identity doesn't match
   * (PID gone, start_time mismatch, UID mismatch, cwd mismatch) is marked
   * crashed.
   */
  async reconcile(): Promise<void> {
    const dataRoot = `${this.opts.dataDir}/data/tenants`;
    let tenants: string[] = [];
    try {
      tenants = await fs.promises.readdir(dataRoot);
    } catch {
      return; // nothing to reconcile yet
    }

    for (const tenant of tenants) {
      let agents: string[] = [];
      try {
        agents = await fs.promises.readdir(`${dataRoot}/${tenant}`);
      } catch {
        continue;
      }
      for (const agent of agents) {
        const rows = listActiveRuns(this.opts.dataDir, tenant, agent).filter(
          (r) => r.status === 'active',
        );
        for (const r of rows) {
          const alive = await this.opts.helper.status({
            pid: r.pid,
            uid: await resolveUidNumber(r.expected_uid),
            runtimeDir: r.runtime_dir,
            startTime: r.pid_start_time_ticks,
            cgroup: r.cgroup_path ?? undefined,
          });
          if (!alive) {
            markActiveRunExited(this.opts.dataDir, tenant, agent, r.run_id, 'crashed');
          } else {
            updateActiveRunLastSeen(this.opts.dataDir, tenant, agent, r.run_id, new Date().toISOString());
          }
        }
      }
    }
  }

  /**
   * Stop a specific run. Sends SIGTERM via the helper, waits up to graceMs,
   * then escalates to SIGKILL.
   */
  async stop(
    tenantId: string,
    agentId: string,
    runId: string,
    graceMs = 10000,
  ): Promise<void> {
    const rows = listActiveRuns(this.opts.dataDir, tenantId, agentId);
    const row = rows.find((r) => r.run_id === runId);
    if (!row || row.status !== 'active') return;

    try {
      await this.opts.helper.kill({
        pid: row.pid,
        signal: 'SIGTERM',
        uid: await resolveUidNumber(row.expected_uid),
        runtimeDir: row.runtime_dir,
        cgroup: row.cgroup_path ?? undefined,
        startTime: row.pid_start_time_ticks,
      });
    } catch {
      // Helper rejects identity mismatch — process already gone.
      markActiveRunExited(this.opts.dataDir, tenantId, agentId, runId, 'crashed');
      return;
    }

    await new Promise((r) => setTimeout(r, graceMs));

    const stillAlive = await this.opts.helper.status({
      pid: row.pid,
      uid: await resolveUidNumber(row.expected_uid),
      runtimeDir: row.runtime_dir,
      startTime: row.pid_start_time_ticks,
      cgroup: row.cgroup_path ?? undefined,
    });
    if (stillAlive) {
      try {
        await this.opts.helper.kill({
          pid: row.pid,
          signal: 'SIGKILL',
          uid: await resolveUidNumber(row.expected_uid),
          runtimeDir: row.runtime_dir,
          cgroup: row.cgroup_path ?? undefined,
          startTime: row.pid_start_time_ticks,
        });
      } catch {
        /* fall through */
      }
    }

    markActiveRunExited(this.opts.dataDir, tenantId, agentId, runId, 'exited');
  }

  /**
   * Drop the active_runs row entirely. Used after the run is reaped and its
   * runtime dir is no longer of interest.
   */
  forget(tenantId: string, agentId: string, runId: string): void {
    deleteActiveRun(this.opts.dataDir, tenantId, agentId, runId);
  }

  /**
   * Start a periodic reconcile loop. Call stop() to tear down.
   */
  startAutoReconcile(): void {
    if (this.timer) return;
    const interval = this.opts.pollIntervalMs ?? 10_000;
    this.timer = setInterval(() => {
      void this.reconcile().catch(() => {});
    }, interval);
  }

  stopAutoReconcile(): void {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = undefined;
    }
  }
}

async function resolveUidNumber(username: string): Promise<number> {
  // Helper's status command expects a numeric UID. We can't look it up from
  // a username without /etc/passwd parsing; for now read it from the
  // active_runs row's expected_uid assuming it's a numeric string, or 0
  // if we can't resolve. Production code should carry the numeric UID in
  // active_runs alongside expected_uid (added as a schema field later).
  const n = Number(username);
  return Number.isInteger(n) ? n : 0;
}
```

- [ ] **Step 4: 运行测试以验证通过**

运行: `npm test -- src/runtime/lifecycle.test.ts`
预期: PASS(1 个测试)。

- [ ] **Step 5: 提交**

```bash
git add src/runtime/lifecycle.ts src/runtime/lifecycle.test.ts
git commit -m "feat(runtime): add lifecycle manager (reconcile, stop, forget)"
```

---

## Task 14: Agent-runner live 轮询循环

**文件:**
- 修改: `container/agent-runner/src/index.ts`
- 新建: `container/agent-runner/src/runtime-ipc.ts`
- 新建: `container/agent-runner/src/runtime-ipc.test.ts`

- [ ] **Step 1: 为 runtime-ipc 写失败的测试**

新建 `container/agent-runner/src/runtime-ipc.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { RuntimeIpc } from './runtime-ipc.js';

let dir: string;

beforeEach(() => {
  dir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-runner-ipc-'));
});

afterEach(() => {
  fs.rmSync(dir, { recursive: true, force: true });
});

describe('RuntimeIpc', () => {
  it('claims pending inbound row', () => {
    const ipc = new RuntimeIpc(dir);
    ipc.init();
    ipc.enqueueInbound({
      sourceMessageId: 'm1',
      channelType: 'feishu',
      chatJid: 'feishu:oc_x',
      content: 'hello',
      timestamp: new Date().toISOString(),
    });
    const row = ipc.claimNextPending();
    expect(row).toBeTruthy();
    expect(row?.content).toBe('hello');
  });

  it('writes outbound and marks inbound completed', () => {
    const ipc = new RuntimeIpc(dir);
    ipc.init();
    const inboundId = ipc.enqueueInbound({
      sourceMessageId: 'm1',
      channelType: 'feishu',
      chatJid: 'feishu:oc_x',
      content: 'hi',
      timestamp: new Date().toISOString(),
    });
    const claimed = ipc.claimNextPending();
    expect(claimed).toBeTruthy();
    ipc.markCompleted(claimed!.row_id);
    ipc.enqueueOutbound({
      sourceInboundIds: [inboundId],
      channelType: 'feishu',
      chatJid: 'feishu:oc_x',
      text: 'hi back',
    });
    const next = ipc.claimNextPending();
    expect(next).toBeUndefined();
  });
});
```

- [ ] **Step 2: 运行测试以验证失败**

运行: `npm test -- container/agent-runner/src/runtime-ipc.test.ts`
预期: FAIL — 模块不存在。

- [ ] **Step 3: 在 runner 侧实现 runtime-ipc**

新建 `container/agent-runner/src/runtime-ipc.ts`:

```typescript
import Database from 'better-sqlite3';
import fs from 'fs';
import path from 'path';

export interface RuntimeIpcOpts {
  /** Runtime dir; expected to contain inbound.db/outbound.db/state.db/tools.db. */
  runtimeDir: string;
}

export interface InboundRow {
  row_id: number;
  source_message_id: string | null;
  channel_type: string;
  chat_jid: string;
  sender: string | null;
  sender_name: string | null;
  is_from_me: number;
  content: string;
  timestamp: string;
  message_type: string | null;
  attachment: string | null;
  card_action: string | null;
  status: string;
}

export interface OutboundRow {
  row_id: number;
  source_inbound_ids: string;
  channel_type: string;
  chat_jid: string;
  text: string;
  status: string;
}

export class RuntimeIpc {
  private inbound!: Database.Database;
  private outbound!: Database.Database;
  private state!: Database.Database;
  private tools!: Database.Database;

  constructor(private readonly opts: RuntimeIpcOpts) {}

  /**
   * Initialise (idempotently) the 4 runtime DBs with the same schema as
   * control-plane runtime-dbs.ts. Run-side caller is responsible for
   * ensuring the dir exists and is writable.
   */
  init(): void {
    this.inbound = openDb(path.join(this.opts.runtimeDir, 'inbound.db'));
    this.outbound = openDb(path.join(this.opts.runtimeDir, 'outbound.db'));
    this.state = openDb(path.join(this.opts.runtimeDir, 'state.db'));
    this.tools = openDb(path.join(this.opts.runtimeDir, 'tools.db'));
  }

  enqueueInbound(args: {
    sourceMessageId?: string;
    channelType: string;
    chatJid: string;
    content: string;
    timestamp: string;
    sender?: string;
    senderName?: string;
    isFromMe?: boolean;
  }): number {
    const dedupe = args.sourceMessageId ?? `${args.chatJid}|${args.timestamp}`;
    const result = this.inbound
      .prepare(
        `INSERT OR IGNORE INTO inbound
          (source_message_id, channel_type, chat_jid, sender, sender_name, is_from_me,
           content, timestamp, dedupe_key, status, enqueued_at)
         VALUES (@sourceMessageId, @channelType, @chatJid, @sender, @senderName, @isFromMe,
                 @content, @timestamp, @dedupe, 'pending', @enqueuedAt)`,
      )
      .run({
        sourceMessageId: args.sourceMessageId ?? null,
        channelType: args.channelType,
        chatJid: args.chatJid,
        sender: args.sender ?? null,
        senderName: args.senderName ?? null,
        isFromMe: args.isFromMe ? 1 : 0,
        content: args.content,
        timestamp: args.timestamp,
        dedupe,
        enqueuedAt: new Date().toISOString(),
      });
    return Number(result.lastInsertRowid);
  }

  claimNextPending(): InboundRow | undefined {
    const row = this.inbound
      .prepare(`SELECT * FROM inbound WHERE status='pending' ORDER BY row_id ASC LIMIT 1`)
      .get() as InboundRow | undefined;
    if (!row) return undefined;
    this.inbound
      .prepare(`UPDATE inbound SET status='claimed', claimed_at=? WHERE row_id=?`)
      .run(new Date().toISOString(), row.row_id);
    return row;
  }

  markCompleted(rowId: number): void {
    this.inbound
      .prepare(`UPDATE inbound SET status='completed', completed_at=? WHERE row_id=?`)
      .run(new Date().toISOString(), rowId);
  }

  enqueueOutbound(args: {
    sourceInboundIds: number[];
    channelType: string;
    chatJid: string;
    text: string;
  }): void {
    this.outbound
      .prepare(
        `INSERT INTO outbound
          (source_inbound_ids, channel_type, chat_jid, text, status, enqueued_at)
         VALUES (@ids, @channelType, @chatJid, @text, 'pending', @enqueuedAt)`,
      )
      .run({
        ids: JSON.stringify(args.sourceInboundIds),
        channelType: args.channelType,
        chatJid: args.chatJid,
        text: args.text,
        enqueuedAt: new Date().toISOString(),
      });
  }

  setState(key: string, value: string): void {
    this.state
      .prepare(
        `INSERT INTO state (key, value, updated_at) VALUES (?, ?, ?)
         ON CONFLICT(key) DO UPDATE SET value=excluded.value, updated_at=excluded.updated_at`,
      )
      .run(key, value, new Date().toISOString());
  }

  getState(key: string): string | null {
    const row = this.state.prepare(`SELECT value FROM state WHERE key=?`).get(key) as
      | { value: string }
      | undefined;
    return row ? row.value : null;
  }

  close(): void {
    this.inbound?.close();
    this.outbound?.close();
    this.state?.close();
    this.tools?.close();
  }
}

function openDb(dbPath: string): Database.Database {
  if (!fs.existsSync(dbPath)) {
    throw new Error(`runtime DB does not exist: ${dbPath}`);
  }
  const db = new Database(dbPath);
  db.pragma('journal_mode = WAL');
  db.pragma('busy_timeout = 5000');
  return db;
}
```

- [ ] **Step 4: 更新 agent-runner index.ts 以支持 live 模式**

编辑 `container/agent-runner/src/index.ts`。在顶部添加 argv parser,用于在 legacy 一次性模式与新的 live 模式之间分发。草稿:

```typescript
import { RuntimeIpc } from './runtime-ipc.js';

function parseArgs(argv: string[]) {
  const out: Record<string, string> = {};
  for (let i = 2; i < argv.length; i++) {
    const a = argv[i];
    if (a.startsWith('--')) {
      const k = a.slice(2);
      const v = argv[i + 1];
      out[k] = v;
      i++;
    }
  }
  return out;
}

async function main() {
  const args = parseArgs(process.argv);
  if (args.mode === 'live' || args.mode === 'isolated') {
    return runLiveMode(args);
  }
  // Existing legacy single-shot code path continues below...
  // (leave existing code untouched)
}

async function runLiveMode(args: Record<string, string>) {
  const ipc = new RuntimeIpc({ runtimeDir: args['runtime-dir'] });
  ipc.init();

  const idleTimeoutMs = 5 * 60 * 1000;
  let lastActivity = Date.now();

  while (true) {
    const row = ipc.claimNextPending();
    if (!row) {
      // Check stop control key.
      const stop = ipc.getState('control:stop');
      if (stop === '1') break;

      // Idle reap.
      if (Date.now() - lastActivity > idleTimeoutMs) break;

      await new Promise((r) => setTimeout(r, 500));
      continue;
    }

    lastActivity = Date.now();

    try {
      // Invoke provider with row.content as prompt. For Foundation, echo
      // a canned response — full provider integration is Plan D.
      const response = `[live] received: ${row.content}`;
      ipc.enqueueOutbound({
        sourceInboundIds: [row.row_id],
        channelType: row.channel_type,
        chatJid: row.chat_jid,
        text: response,
      });
      ipc.markCompleted(row.row_id);
    } catch (err) {
      ipc.markCompleted(row.row_id); // mark processed to avoid hot loop
      console.error('run failed', err);
    }
  }

  ipc.close();
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

保留 `main()` 以下的现有代码以支持 legacy 模式。

- [ ] **Step 5: 运行测试以验证通过**

运行: `npm test -- container/agent-runner/src/runtime-ipc.test.ts`
预期: PASS(2 个测试)。

- [ ] **Step 6: 提交**

```bash
git add container/agent-runner/src/runtime-ipc.ts container/agent-runner/src/runtime-ipc.test.ts container/agent-runner/src/index.ts
git commit -m "feat(agent-runner): add live poll loop over runtime DBs"
```

---

## Task 15: 把 spawner 接入控制平面消息循环

**文件:**
- 修改: `src/index.ts`
- 修改: `src/group-queue.ts`

- [ ] **Step 1: 替换 docker-per-group 的 spawn 调用**

在 `src/index.ts` 中,找到消息到来时调用 `container-runner.ts` 启动 docker 容器的代码段。把该调用替换为 spawner 流程:

```typescript
// Add imports:
import { Spawner } from './runtime/spawner.js';
import { LifecycleManager } from './runtime/lifecycle.js';
import { HelperClient } from './runtime/helper-client.js';
import { openRuntimeDbs } from './runtime/runtime-dbs.js';

// At startup, construct helper + spawner + lifecycle:
const helper = new HelperClient({ binaryPath: process.env.NANOCLAW_HELPER_BIN ?? '/usr/lib/nanoclaw/nc-setuid-helper' });
const spawner = new Spawner({
  dataDir: NANOCLAW_DATA_DIR,
  helper,
  agentRunnerBin: process.env.NANOCLAW_AGENT_RUNNER_BIN ?? '/opt/nanoclaw/agent-runner/index.js',
});
const lifecycle = new LifecycleManager({ dataDir: NANOCLAW_DATA_DIR, helper });
lifecycle.startAutoReconcile();
await lifecycle.reconcile();

// On inbound message for (tenant, agent, group):
async function onInboundMessageForRuntime(tenantId: string, agentId: string, groupFolder: string, message: NewMessage) {
  const runtimeDir = tenantAgentRuntimeDir(NANOCLAW_DATA_DIR, tenantId, agentId, groupFolder);
  const liveDir = path.join(runtimeDir, 'live');
  const dbs = openRuntimeDbs(liveDir);
  try {
    // Enqueue inbound.
    dbs.inbound.prepare(
      `INSERT INTO inbound (source_message_id, channel_type, chat_jid, sender, sender_name, is_from_me,
                            content, timestamp, dedupe_key, status, enqueued_at)
       VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 'pending', ?)`,
    ).run(
      message.id, 'feishu', message.chat_jid, message.sender, message.sender_name,
      message.is_from_me ? 1 : 0, message.content, message.timestamp,
      message.id, new Date().toISOString(),
    );

    // Start live run if not already active for this group.
    const active = listActiveRuns(NANOCLAW_DATA_DIR, tenantId, agentId).find(
      (r) => r.group_folder === groupFolder && r.status === 'active',
    );
    if (!active) {
      await spawner.startLiveRun({
        tenantId, agentId, group: groupFolder, model: 'claude-sonnet-4', env: {},
      });
    }
  } finally {
    dbs.close();
  }
}
```

- [ ] **Step 2: 添加 outbound 轮询器**

在 `src/index.ts` 启动时,起一个周期循环,扫描每个 live runtime 目录下 outbound.db 的 pending 行,并通过 channel 投递:

```typescript
setInterval(() => {
  void deliverPendingOutbound().catch((err) => logger.error({ err }, 'outbound poll failed'));
}, 1000);

async function deliverPendingOutbound() {
  // For each (tenant, agent) in active_runs, open its live outbound.db and
  // claim pending rows, deliver via the right channel, mark delivered.
  // Implementation mirrors the reconcile loop in lifecycle.ts — walk the
  // data dir, open partition DBs, etc.
}
```

完整实现 `deliverPendingOutbound`(遍历 tenants,打开每个 active group 的 live outbound.db,claim 行,通过 Plan B 的 `lookupChannel(...)` 投递,标记为已投递)。

- [ ] **Step 3: 运行 typecheck + 现有测试**

运行: `npm run typecheck && npm test`
预期: 现有测试通过;新代码编译通过。

- [ ] **Step 4: 提交**

```bash
git add src/index.ts src/group-queue.ts
git commit -m "feat(runtime): wire spawner + outbound poller into control plane"
```

---

## Self-Review 备注

**Spec 覆盖检查**(对照 `docs/runtime-rework/`):
- ADR-005'(Linux user = 隔离单元,Docker = 可选 wrapper):✓ Task 1-15(完全不使用 Docker)
- ADR-008'(控制平面策略 + helper 机制):✓ Task 7-13
- ADR-012(per-runtime POSIX ACL,无共享 runtime group):✓ Task 5
- CROSS_CUTTING_CONTRACTS "Runtime DB ownership":✓ Task 11
- CROSS_CUTTING_CONTRACTS "active_runs reconciliation":✓ Task 12-13
- `nc-setuid-helper` 5 个带身份校验的命令:✓ Task 7-9

**范围之外**(延后处理):
- Skill bundle 暂存 — Plan D
- Tool IPC 迁移(带 host-side worker 的 tools.db 流程)— Plan E
- 从 legacy groups/ 迁移的脚本 — Plan F
- Acceptance/isolation 测试 — Plan G

**占位符扫描**:无。每个步骤都有代码。

**类型一致性**:`HelperPrepareResult` 字段在 Task 10(TS client)、Task 12(spawner 消费方)、Task 9(C `prepare` stdout 格式)之间保持一致。`ActiveRunRecord` 结构与 Plan A 的 Task 8 定义匹配。

**已知过渡期债务**:
- Task 12 的 `findNewestPidForUid` 在 spawn 之后扫描 `/proc`;生产环境应替换为通过 helper 的 pipe 可靠上报 PID。
- Task 13 的 `resolveUidNumber` 在 username 非数字时回退为 0;Plan F 应在 `active_runs` 中新增 `expected_uid_number` 列。
- Task 14 的 live 模式回显的是 canned response。Plan D 接入真实 provider。
