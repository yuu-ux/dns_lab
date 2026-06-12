---
marp: true
theme: default
paginate: true
---

# DNS名前解決エラーのデバッグ

## dig で障害を切り分ける

Mastering TCP/IP 勉強会

---

# 今日のゴール

- `dig` コマンドの使い方と出力の読み方を覚える
- 2種類の代表的なDNSエラーを体験する
  - **NXDOMAIN**: レコードが存在しない
  - **SERVFAIL**: サーバー側の障害
- 「名前解決できない」ときの切り分け手順を身につける

---

# そもそもDNSとは

- **Domain Name System** = 名前とIPアドレスの対応を管理する仕組み
- `www.google.com` → `142.250.xxx.xxx` に変換してくれる（名前解決）
- インターネットの **電話帳** のようなもの

なぜ必要？
- 人間は `www.google.com` が覚えやすい
- コンピュータは `142.250.196.99` でないと通信できない
- DNSがこのギャップを埋める

---

# DNSは階層構造

```
.（ルート）
├── com.
│   ├── google.com.
│   │   └── www.google.com.  → 142.250.xxx.xxx
│   └── amazon.com.
├── jp.
│   └── co.jp.
│       └── amazon.co.jp.
└── test.                    ← 今日使うテスト用TLD
    └── example.test.
```

- ドメイン名は **右から左** に読む（`.` → `com` → `google` → `www`）
- 各階層に **担当のDNSサーバー** がいる

---

# 権威サーバーとフルリゾルバ

**権威サーバー（Authoritative Server）**
- 「自分が管理するゾーンのレコード」を持っているサーバー
- 例: `google.com` の権威サーバーは `www` や `mail` のIPアドレスを知っている
- 階層ごとに別の権威サーバーがいる（ルート、`.com`、`google.com`、…）

**フルリゾルバ（Full Resolver / キャッシュDNSサーバー）**
- アプリの代わりに、権威サーバーを **階層を辿って** 問い合わせてくれるサーバー
- 結果をキャッシュして次回以降は速く返す
- 例: `8.8.8.8`（Google）、`1.1.1.1`（Cloudflare）

---

# DNS名前解決の流れ

```
あなたのアプリ
    │  「api.example.com のIPは？」
    ▼
フルリゾルバ（キャッシュDNSサーバー）
    │  知らなければ上流に問い合わせ
    ▼
権威DNSサーバー
    │  「192.0.2.1 だよ」
    ▼
フルリゾルバ → アプリに回答
```

- アプリは直接権威サーバーに聞かない（フルリゾルバが代行）
- フルリゾルバが壊れていると → **SERVFAIL**
- 権威サーバーにレコードがないと → **NXDOMAIN**

---

# dig コマンドの読み方

```
$ dig @8.8.8.8 www.google.com

;; ->>HEADER<<-
;;   status: NOERROR        ← ★ ここが最重要
;;   flags: qr rd ra

;; ANSWER SECTION:          ← ★ 実際の回答
www.google.com.  300  IN  A  142.250.xxx.xxx

;; Query time: 12 msec      ← 応答時間
```

| 注目ポイント | 意味 |
|---|---|
| `status` | NOERROR / NXDOMAIN / SERVFAIL |
| `ANSWER SECTION` | 解決結果（空なら失敗） |
| `Query time` | 応答速度 |

---

# DNSエラーコード一覧

| コード | 意味 | よくある原因 |
|---|---|---|
| **NOERROR** | 正常 | — |
| **NXDOMAIN** | ドメインが存在しない | typo、レコード未登録 |
| **SERVFAIL** | サーバーエラー | 上流障害、ネットワーク障害 |
| **REFUSED** | 拒否された | 再帰問い合わせ無効、ACL |

今日は **NXDOMAIN** と **SERVFAIL** を体験します

---

# ハンズオンで体験しましょう

```bash
# 環境起動
docker compose up -d

# 手順書を開いて進めてください
# handson/handson_guide.md
```
