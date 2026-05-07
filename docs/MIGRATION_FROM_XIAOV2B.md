# 从 xiaov2board 迁移到 v2board-duplication

> 适用场景：生产环境跑的是 [`null404-0/xiaov2board`](https://github.com/null404-0/xiaov2board)（原版 / 含"专属节点"功能），需要切到精简后的 [`null404-0/v2board-duplication`](https://github.com/null404-0/v2board-duplication)（魔改版 / 移除了专属节点）。

## TL;DR

- 两个仓库**只有一项功能差异**：用户专属节点（dedicated nodes / `userAssign`）。原版有，魔改版砍掉了。
- 数据库结构**只有一张表的差异**：`v2_server_user`。原版独有，魔改版没用到。
- 用户表 / 订单表 / 套餐表 / 服务器表 / 订阅 / 支付 / UniProxy 协议**完全一致**。
- 因此**不需要做数据迁移、字段映射、ID 重映射**。本质上就是「换代码 → 重启」。
- 唯一风险点：如果你正在使用专属节点功能，切换后这些节点会回退到「同 group 全员可见」。

---

## 1. 仓库间到底差什么（精确清单）

`diff -rq` 全量对比的结果（去掉 `.git`）：

```
Only in orig/app/Http/Controllers/V1/Admin/Server: UserAssignController.php
Only in orig/app/Models: ServerUser.php
Only in orig/resources/views: dedicated-nodes.blade.php
Files differ: app/Http/Routes/V1/AdminRoute.php
Files differ: app/Http/Controllers/V1/Server/UniProxyController.php
Files differ: app/Services/ServerService.php
Files differ: routes/web.php
Files differ: database/install.sql
Files differ: database/update.sql
```

| 文件 | 原版 (xiaov2board) | 魔改版 (v2board-duplication) |
|---|---|---|
| `app/Models/ServerUser.php` | ✅ 模型，对应 `v2_server_user` 表 | ❌ 删除 |
| `app/Http/Controllers/V1/Admin/Server/UserAssignController.php` | ✅ 后台 API：`fetch` / `save` / `searchUsers` | ❌ 删除 |
| `resources/views/dedicated-nodes.blade.php` | ✅ 后台管理页面 | ❌ 删除 |
| `app/Http/Routes/V1/AdminRoute.php` | 注册 `/server/userAssign/*` 三条路由 | 不注册 |
| `routes/web.php` | 注册 `/<secure_path>/dedicated-nodes` 页面路由 | 不注册 |
| `app/Services/ServerService.php` | 节点可见性逻辑：节点被任何人专属过 → **只对被分配的用户可见**；否则走 `group_id` | 纯 `group_id` 控制可见性 |
| `app/Http/Controllers/V1/Server/UniProxyController.php` | 调用 `getAvailableUsers($group, $serverId, $type)` | 调用 `getAvailableUsers($group)` |
| `database/install.sql` / `database/update.sql` | 包含 `CREATE TABLE v2_server_user` | 不包含 |

`v2_server_user` 表结构（仅供参考）：

```sql
CREATE TABLE `v2_server_user` (
  `server_id`   int(10) unsigned NOT NULL,
  `server_type` varchar(32)      NOT NULL,
  `user_id`     int(10) unsigned NOT NULL,
  PRIMARY KEY (`server_id`, `server_type`, `user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

> 其它一切（`v2_user`, `v2_order`, `v2_plan`, `v2_server_*`, 支付、工单、邀请码、流量统计…）两边一字不差。

---

## 2. 迁移前预检查（生产环境）

```sql
-- 看一下专属节点功能到底有没有被用过
SELECT COUNT(*) AS rows_total FROM v2_server_user;

-- 哪些节点被设过专属、各分了几个用户
SELECT server_type, server_id, COUNT(DISTINCT user_id) AS user_cnt
FROM v2_server_user
GROUP BY server_type, server_id
ORDER BY user_cnt DESC;
```

根据结果分两种情况：

### 情况 A：`v2_server_user` 是空的
随便切，零业务影响。直接走 §4。

### 情况 B：`v2_server_user` 里有数据
**这些节点切到魔改版后会立即对同 group 的所有用户可见。** 三个选择：

1. 接受这个变化，直接切（最省事）。
2. 切之前先把这些节点 `v2_server_*` 表里 `show=0` 隐藏，迁后再人工评估哪些放出来。
3. 不接受 — 那就别迁，或者把这套功能加回魔改版（参考原版那 4 个文件 + service 中的 `isServerVisible` 逻辑 cherry-pick）。

---

## 3. 完整备份（必做）

```bash
# 数据库 — 一致性快照
mysqldump -u<user> -p<pwd> --single-transaction --routines --triggers \
  <db_name> > /backup/v2b-$(date +%F-%H%M).sql

# 代码 + 上传文件 + .env + storage（包含 session/cache/logs）
tar czf /backup/v2b-code-$(date +%F-%H%M).tgz \
  --exclude='vendor' --exclude='node_modules' /www/wwwroot/v2board

# 记下当前的 commit，用于回滚
cd /www/wwwroot/v2board && git rev-parse HEAD > /backup/v2b-commit-$(date +%F).txt
```

---

## 4. 在测试机演练（你已经有这台测试机，按这个流程过一遍）

1. 把生产 `mysqldump` 导入测试机数据库。
2. 测试机上跑魔改版代码，指向这份 dump。
3. **必跑用例**：
   - [ ] 用户登录、注册、找回密码
   - [ ] 订阅 URL（clash / shadowrocket / sing-box / v2rayN）拉取节点
   - [ ] 各类型节点连接测试（vless / vmess / trojan / shadowsocks / hysteria / tuic / anytls / v2node — 凡是你在用的）
   - [ ] 下单、扫码支付、对账
   - [ ] 后台用户列表、订单列表、节点列表、套餐管理
   - [ ] 节点服务端调用 `/api/v1/server/UniProxy/user` 拉用户列表（最易出问题的接口）
   - [ ] 工单 / 邀请 / 公告 / 邮件队列

---

## 5. 生产切换（建议在维护窗口内）

```bash
cd /www/wwwroot/v2board

# 5.1 停服
php -c cli-php.ini webman.php stop

# 5.2 切代码源
git remote set-url origin https://github.com/null404-0/v2board-duplication.git
git fetch origin
git checkout master
git reset --hard origin/master   # 注意：会丢弃任何本地未提交改动，务必确认 5.2 之前已 commit/备份

# 5.3 装依赖 + 跑 v2board:update（仓库自带脚本）
bash update.sh
# update.sh 内部干的事：
#   - 重新拉 composer.phar
#   - composer update
#   - 装 adapterman（PHP 8+）
#   - php artisan v2board:update   ← 会跑 database/update.sql 里"对你当前数据库还没执行过"的语句
#                                     魔改版的 update.sql 不含 CREATE TABLE v2_server_user，
#                                     所以对已有库零修改、零破坏

# 5.4 清缓存
php artisan cache:clear
php artisan config:clear
php artisan route:clear

# 5.5 启动
php -c cli-php.ini start.php start -d
```

> ⚠️ `.env` 不在 git 管理范围，`git reset --hard` **不会动它**。`storage/`、`public/uploads/` 同样不会被覆盖。但请用 `git status -uall` 自查一遍，避免有人在生产改过被跟踪文件。

---

## 6. 切换后验证（按顺序）

```bash
# 进程在跑？
ps -ef | grep -E 'webman|workerman'

# 日志在不在哭？
tail -f storage/logs/$(date +%F).log
tail -f storage/logs/laravel.log

# 关键接口
curl -I https://你的域名/api/v1/guest/comm/config
curl https://你的域名/api/v1/server/UniProxy/user?token=<节点token>&node_id=<id>&node_type=<type>
```

业务验证（同 §4 的清单，在生产再跑一遍）。

---

## 7. 回滚预案

如果 1 小时内没跑通：

```bash
cd /www/wwwroot/v2board
php -c cli-php.ini webman.php stop

git remote set-url origin https://github.com/null404-0/xiaov2board.git
git fetch origin
git reset --hard <旧 commit，从 §3 备份的 v2b-commit 文件里取>
bash update.sh

php -c cli-php.ini start.php start -d
```

数据库**不需要还原**，因为整个迁移过程没有对它做任何破坏性变更（魔改版的 `update.sql` 是原版的子集）。

---

## 8. 善后清理（可选，建议两周稳定后再做）

```sql
-- 留个时间戳归档表，再删
CREATE TABLE v2_server_user_archive_$(date '+%Y%m%d')
  AS SELECT * FROM v2_server_user;

DROP TABLE v2_server_user;
```

---

## 9. 常见踩坑

- **`php artisan v2board:update` 报错**：通常是 MySQL 用户对当前 DB 没有 `ALTER` 权限，或之前手动改过表结构。把报错贴出来对照 `database/update.sql` 看哪一条挂了。
- **节点上线后用户拉不到流量**：检查节点的 `token` 是否还在 `v2_server_*` 表里、节点服务端日志中 `/api/v1/server/UniProxy/user` 的返回。魔改版改了 `getAvailableUsers` 签名但**对外接口路径不变**。
- **`v2_server_user` 表残留导致疑惑**：魔改版根本不读这张表，留着不影响业务，只是占空间。
- **后台进 `/dedicated-nodes` 404**：正常，这个页面被砍了。

---

## 附：单条命令拉出两个仓库的完整差异（任何时候自查）

```bash
git clone --depth=1 https://github.com/null404-0/xiaov2board.git orig
git clone --depth=1 https://github.com/null404-0/v2board-duplication.git mod
diff -rq orig mod | grep -v '\.git'
```
