---
title: "UbuntuでCodexのsandbox初期化エラーを解消する"
date: 2026-09-16
tags:
  - Codex
  - Ubuntu
  - AppArmor
  - sandbox
summary: "AppArmorのuser namespace制限によってCodexのsandboxを初期化できない問題を解消した記録"
---

## ご相談

UbuntuでCodexを起動すると、sandboxの初期化時に次のエラーが発生するという相談がありました。

```text
bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted
```

## 調べたこと

Codexのsandboxは、Ubuntu上では `bwrap` を使って隔離環境を作ります。

user namespaceとネットワークnamespaceを作れるか、次のコマンドで確認してもらいました。

```bash
unshare -Urn sh -c 'ip link set lo up && ip addr add 127.0.0.1/8 dev lo'
```

結果は次のとおりでした。

```text
unshare: write failed /proc/self/uid_map: Operation not permitted
```

関連する設定を確認すると、user namespace自体は有効でしたが、AppArmorによる制限が有効になっていました。

```text
kernel.unprivileged_userns_clone = 1
kernel.apparmor_restrict_unprivileged_userns = 1
user.max_user_namespaces = 116132
```

## 解決

AppArmorのuser namespace制限を一時的に無効化しました。

```bash
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```

その後、`unshare` でloopbackを操作できるようになり、Codexから `pwd` を実行してsandboxが正常に初期化されることを確認できました。

再起動後も設定を維持する場合は、`/etc/sysctl.d/99-codex-userns.conf` に次の内容を保存します。

```conf
kernel.apparmor_restrict_unprivileged_userns=0
```

設定を反映します。

```bash
sudo sysctl --system
```

今回はこれで無事、通常のsandbox付きでCodexを使えるようになりました。

## 補足

この設定はシステム全体のセキュリティ方針に関係します。共有端末や管理対象端末では、恒久化する前に管理者へ確認します。
