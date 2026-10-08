# 港GQX 规则集 — 维护手册

> 港/美股券商分流规则（富途 / Moomoo / 长桥 / 老虎）
> 来源: [blackmatrix7/ios_rule_script#1687](https://github.com/blackmatrix7/ios_rule_script/issues/1687)
> 路由器: iStoreOS + OpenClash 0.47.097（Mihomo 内核），管理地址 `192.168.100.1`

---

## 一、架构总览（三块东西，各管一摊）

```
┌─ GitHub 仓库（本仓库）────────────────────────────┐
│  港GQX.yaml  ← 规则内容本体（域名 + IP 段）          │
└──────────────┬───────────────────────────────────┘
               │ http 拉取（走代理，每 24h 自动更新）
┌──────────────▼─ 路由器 ───────────────────────────┐
│ ① /etc/openclash/custom/openclash_custom_rules.list │
│    · rule-providers: 定义 GQX 规则集（URL 在这里）   │
│    · rules: RULE-SET,GQX,港GQX 及其他自定义规则      │
│    · ★ 浏览器可编辑：LuCI→OpenClash→覆写设置→规则设置→自定义规则 │
│ ② /etc/openclash/custom/openclash_custom_overwrite.sh│
│    · 创建策略组 PROXY / AUTO / 港GQX                 │
│    · 补回被组名校验误杀的自定义规则                   │
│    · 修正 dns.fallback 走代理                        │
│    · LuCI→覆写设置→开发者选项 可编辑                 │
└──────────────────────────────────────────────────┘
```

**职责边界**：改规则内容 → 只动 GitHub 的 `港GQX.yaml`；改分流指向/加减路由规则 → LuCI「自定义规则」页面；策略组定义 → 覆写脚本（几乎不用动）。

---

## 二、日常维护操作

### 2.1 增删券商域名/IP（最常见）

1. 编辑本仓库 `港GQX.yaml` 的 `payload` 列表（保持缩进两格，格式：`  - DOMAIN-SUFFIX,xxx.com`）
2. commit + push
3. 完成。路由器每 24 小时自动拉新；要立即生效见 [2.4](#24-让路由器立即拉取新规则)

### 2.2 加一条分流规则（某个域名走代理/直连）

浏览器打开 **LuCI → 服务 → OpenClash → 覆写设置 → 规则设置 → 自定义规则**，在 `rules:` 列表里加一行：

```yaml
- DOMAIN-SUFFIX,example.com,PROXY    # 走代理
- DOMAIN-SUFFIX,example.com,DIRECT   # 走直连
```

保存&应用即生效。**规则自上而下匹配，命中即停**——注意别放在会提前命中的兜底规则后面。

> 注意：指向 `PROXY` / `AUTO` / `港GQX` 的规则由覆写脚本自动"补回"（见第四节），正常可用；指向其他自建组名的规则需同步改覆写脚本第 135 行过滤器。

### 2.3 切换券商流量走向

浏览器打开代理面板 `http://192.168.100.1:9090/ui` → 「代理」页 → `港GQX` 组：

- `PROXY`：跟随主代理（默认）
- `AUTO`：自动测速选最快节点
- `DIRECT`：直连（境内网络交易富途时延迟最低，其服务器多为腾讯云国内段）

### 2.4 让路由器立即拉取新规则

面板 → 「配置」/「Providers」页找到 `GQX` 点刷新；或 SSH 执行：

```sh
/etc/init.d/openclash restart
```

---

## 三、策略组说明

| 组名 | 类型 | 作用 | 定义位置 |
|---|---|---|---|
| `PROXY` | select + include-all | 手动选择主代理组。`include-all` 自动收纳订阅全部节点，**换订阅不失效** | 覆写脚本 |
| `AUTO` | url-test + include-all | 自动测速（300s 间隔，容差 50ms） | 覆写脚本 |
| `港GQX` | select [PROXY, AUTO, DIRECT] | 券商流量独立开关 | 覆写脚本 |

> 自定义规则/规则集只允许引用 `PROXY` / `AUTO` / `港GQX` / `DIRECT`，**不要引用订阅自带组名**（如"默认节点"）——那会重新引入"换订阅组名失效"的问题。

---

## 四、覆写脚本修改指南（`openclash_custom_overwrite.sh`）

### 执行时机

`/etc/init.d/openclash restart` 的流水线：复制订阅 → DNS 处理 → 规则处理 → 官方覆写 → **执行本脚本（最后一步）**。脚本通过 `$1` 接收最终配置文件路径，所有修改用 ruby 对该 YAML 做手术。

### 建组核心逻辑（第 113–147 行 ruby 块）

```ruby
unless names.include?('PROXY') then     # ← 幂等：已存在就跳过
  groups.unshift({'name'=>'PROXY', 'type'=>'select', 'include-all'=>true});
end;
```

- 加组：复制一段 `unless ... end;`，改组名和 Hash
- 改属性：直接改 Hash（如 `interval:300` → `600`）
- 删组：删对应 `unless` 块

### 两处必须联动的坑

1. **第 135 行过滤器**：`['PROXY','AUTO','港GQX'].include?(...)`。OpenClash 在流水线早期会校验规则指向的组是否存在，此时自建组还没创建，规则会被静默删除；这段过滤器在最后把指向自建组的规则补回。**新建/改名组时必须把组名加进这个数组**。
2. **自定义规则文件里的引用**：组改名后，`rules:` 里所有 `,旧组名` 结尾的行要同步改。

### ruby 语法注意

行尾分号、单引号 Hash（`'key'=>'val'`）、`end;` 缺一个整块就静默失效。改完看日志：

```sh
grep -a 'Failed' /tmp/openclash.log | tail    # 有输出 = 脚本出错了
```

---

## 五、生效与验证

```sh
# SSH 到路由器（192.168.100.1）
/etc/init.d/openclash restart && sleep 25       # 注意 reload 无效，必须 restart

grep -c '港GQX' /etc/openclash/BYG-GF.yaml       # ≥2 = 组和规则已注入
sed -n "$(grep -n '^  fallback:' /etc/openclash/BYG-GF.yaml | cut -d: -f1),+1p" /etc/openclash/BYG-GF.yaml
# 应为: https://dns.google/dns-query#PROXY
grep -a 'Failed' /tmp/openclash.log | tail       # 空 = 无报错
grep -a 'RuleSet(GQX)' /tmp/openclash.log | tail # 有命中记录 = 规则链路通
```

面板确认：`港GQX` 组存在且可切换；Providers 里 `GQX` 规则数 = 仓库 `港GQX.yaml` 的条数。

---

## 六、故障排查速查

| 症状 | 排查顺序 |
|---|---|
| 券商域名没走港GQX组 | ① 面板「规则」页第一条是否 `RULE-SET,GQX,港GQX` ② `grep -a 'Skiped' /tmp/openclash.log` 看是否被组名校验删了（应被钩子补回）③ 规则前是否有更早命中的规则 |
| provider 规则数对不上 | `ls -la /etc/openclash/rule_provider/GQX.yaml` 看文件时间；删掉它再 restart 强制重新下载 |
| 全部域名解析失败 | 看 fallback 是否还是 `#PROXY`（覆写脚本第 141 行）；`grep -a 'Failed' /tmp/openclash.log` |
| 改了规则不生效 | 是否忘了 push？provider 有 24h 缓存（见 2.4 立即刷新）；restart 而不是 reload |
| 覆写脚本改挂了 | 用备份回滚（见下节），再 `restart` |

---

## 七、备份与回滚

**历史备份**（都在路由器上）：

```
/etc/openclash/custom/openclash_custom_rules.list.bak-20261009-0115
/etc/openclash/custom/openclash_custom_overwrite.sh.bak-20261009-0115
/etc/openclash/custom/*.bak-20261008-2230 / 224056
/etc/config/openclash.bak-20261008-222048
```

**改动前先备份**：

```sh
TS=$(date +%m%d-%H%M)
cp /etc/openclash/custom/openclash_custom_overwrite.sh{,.bak-$TS}
cp /etc/openclash/custom/openclash_custom_rules.list{,.bak-$TS}
```

**回滚**：用备份覆盖原文件 → `/etc/init.d/openclash restart`。

---

## 八、OpenClash 升级注意事项

- `/etc/openclash/custom/` 与 `/etc/config/openclash` **不在升级覆盖范围**，脚本和规则都安全
- 但脚本依赖的官方库（`ruby.sh`、`yml_rules_change.sh` 的 rule-providers 合并、组名校验行为）**可能变化**，升级后跑一遍第五节的验证
- 升级前打个包最稳：

```sh
tar -czf /tmp/openclash-custom-$(date +%m%d).tar.gz -C / etc/openclash/custom /etc/config/openclash
```

- 若升级后 rule-providers 合并失效（provider 消失），把 `rule-providers` 定义搬回调用的 ruby 块（历史版本有现成写法，见 git 记录）

---

*创建: 2026-10-09 · 规则来源: issue #1687 · 维护: 只改本仓库的 港GQX.yaml 和路由器两个 custom 文件*
