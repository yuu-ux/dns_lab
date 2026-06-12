# DNS名前解決エラー デバッグ ハンズオン手順書

## 事前準備

```bash
# 障害環境を起動
docker compose up -d

# コンテナが起動していることを確認
docker compose ps
# dns-nxdomain と dns-servfail が running であればOK
```

### dig コマンドとは

`dig` はDNSサーバーに問い合わせるコマンド。「この名前のIPアドレスを教えて」とDNSサーバーに聞ける。

```bash
dig @DNSサーバーのIP -p ポート番号 問い合わせたい名前
```

---

## Step 1: 正常系の確認（ベースライン）

### 観察ポイント（先に確認！）
- `status: NOERROR` が返るはず
- `ANSWER SECTION` にIPアドレスが表示されるはず

### コマンド

```bash
dig @8.8.8.8 www.google.com
```

### 答え合わせ

出力の中で注目すべき箇所:

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
www.google.com.     300  IN  A  142.250.xxx.xxx
```

### dig 出力の読み方

| 項目 | 意味 |
|------|------|
| `status: NOERROR` | 正常に解決できた |
| `flags: qr rd ra` | qr=応答, rd=再帰要求, ra=再帰可能 |
| `ANSWER: 1` | 回答が1件ある |
| `ANSWER SECTION` | 実際のIPアドレスが表示される |
| `Query time` | 応答にかかった時間（ms） |

### 解説
- これが「正常な名前解決」の姿。このあとのStepで異常系と比較する
- `status: NOERROR` + `ANSWER SECTION` にレコードあり = 名前解決成功

---

## Step 2: NXDOMAIN — レコードが存在しない

### 観察ポイント（先に確認！）
- 存在するレコード (`www`) は正常に返るはず
- 存在しないレコード (`app`) は `status: NXDOMAIN` になるはず

### コマンド

```bash
# まず正常なレコードを確認
dig @127.0.0.1 -p 15301 www.example.test

# 次に存在しないレコードを確認
dig @127.0.0.1 -p 15301 app.example.test
```

### 答え合わせ

**www.example.test（正常）:**
```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: xxxxx
;; ANSWER SECTION:
www.example.test.   300  IN  A  192.0.2.1
```

**app.example.test（NXDOMAIN）:**
```
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: xxxxx
;; AUTHORITY SECTION:
example.test.   300  IN  SOA  ns1.example.test. admin.example.test. ...
```

### 確認すべきフィールド

| フィールド | www（正常） | app（NXDOMAIN） |
|---|---|---|
| status | NOERROR | **NXDOMAIN** |
| ANSWER SECTION | A レコードあり | **なし** |
| AUTHORITY SECTION | なし | **SOA レコードが返る** |

### 解説
- **NXDOMAIN** = 「その名前はこのゾーンに存在しない」
- DNSサーバーは動いている。ゾーンも正しく読み込まれている。ただレコードが登録されていないだけ
- AUTHORITY SECTION に SOA レコードが返るのは「この応答をどれくらいキャッシュしてよいか」を伝えるため（ネガティブキャッシュ）
- 実務では: typo、レコード登録忘れ、ゾーンファイルの反映漏れが原因になることが多い

---

## Step 3: SERVFAIL — サーバー側の障害

### 観察ポイント（先に確認！）
- `status: SERVFAIL` が返るはず
- ANSWER SECTION は空のはず
- 同じドメインを別のDNSサーバーに聞くと正常に解決できるはず

### コマンド

```bash
# 壊れたDNSサーバーに問い合わせ（上流のタイムアウトを待つので10秒ほどかかる）
dig @127.0.0.1 -p 15302 www.google.com +time=10

# 同じドメインを正常なDNSサーバー（Google Public DNS）に問い合わせ
dig @8.8.8.8 www.google.com

# サーバーのログを確認
docker logs dns-servfail
```

### 答え合わせ

**壊れたDNSサーバー（SERVFAIL）:**
```
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: xxxxx
;; ANSWER SECTION なし
```

**正常なDNSサーバー（NOERROR）:**
```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: xxxxx
;; ANSWER SECTION:
www.google.com.   300  IN  A  142.250.xxx.xxx
```

**docker logs の出力例:**
```
[ERROR] plugin/errors: 2 www.google.com. A: read udp ...: i/o timeout
```

### 確認すべきフィールド

| フィールド | SERVFAIL | 正常（8.8.8.8） |
|---|---|---|
| status | **SERVFAIL** | NOERROR |
| ANSWER SECTION | **なし** | A レコードあり |

### 解説
- **SERVFAIL** = 「DNSサーバーが処理中にエラーが発生した」
- NXDOMAIN との違い: NXDOMAIN は「レコードが存在しない」という**正常な応答**。SERVFAIL は「サーバーが壊れている」という**異常な応答**
- 切り分けの鍵: **同じドメインを別のDNSサーバーに聞く**。別のサーバーで解決できるなら、ドメインではなくサーバー側の問題
- `docker logs` でサーバー側のエラーログを確認できる。実務では `/var/log/named/` や `journalctl -u named` など
- 実務での原因: 上流DNSサーバーの障害、ネットワーク障害、DNSSEC検証失敗など

---

## まとめ: デバッグ切り分けチェックリスト

「名前解決ができない」と言われたら:

```
1. dig でステータスコードを確認
   └─ status は何？
       ├─ NXDOMAIN → レコードが存在しない（登録忘れ? typo?）
       ├─ SERVFAIL → サーバー側の障害
       │   └─ 別のDNSサーバーで試す → 解決できる？
       │       ├─ YES → そのDNSサーバーの問題
       │       └─ NO  → ドメイン自体の問題（委譲切れなど）
       └─ NOERROR + ANSWER空 → レコードタイプの不一致（NODATA）

2. サーバーのログを確認
   └─ docker logs / /var/log/named/ / journalctl

3. 別のDNSサーバーで比較
   └─ dig @8.8.8.8 / dig @1.1.1.1
```

### NXDOMAIN vs SERVFAIL 比較表

| | NXDOMAIN | SERVFAIL |
|---|---|---|
| 意味 | レコードが存在しない | サーバーエラー |
| サーバーは正常？ | はい | **いいえ** |
| 別のDNSサーバーで同じ結果？ | はい（レコードがないから） | **いいえ**（サーバーの問題だから） |
| よくある原因 | typo, 登録忘れ | 上流障害, ネットワーク障害 |
| 対処 | レコードを登録する | サーバー/ネットワークを修復する |

## 後片付け

```bash
docker compose down
```
