# Level 2: 標準構成(推奨セットアップ)

**目的**: FIPS v0.5.0 on Windows で、**LAN(mDNS 自動発見)** と **グローバル(Nostr リレー仲介 + NAT 越え)** の両方を有効にし、固定の `npub` を持ち、`<npub>.fips` 名であらゆるアプリから通信できる状態を作る。

目標構成図:

```text
あなたの Windows ノード (npub1xxx... = 固定ID)
  ├── LAN内FIPSノード ──── mDNS (_fips._udp.local.) で自動発見 → 直接UDPピア
  ├── test-us01(公開テストメッシュ) ──── 静的UDPピア → メッシュ全体へ到達
  └── 外部の知人ノード ──── Nostrリレー経由でエンドポイント解決
                            + STUN/ホールパンチングで NAT越し直接UDP
```

---

## Step 0: 前提確認

- Windows 10/11 x64、**管理者 PowerShell** で作業
- IPv6 が有効(`ipconfig` で各アダプタのリンクローカル `fe80::` が見えること)
- インストールは [Level 1](LEVEL1-quickstart.md) Step 1–3 と同じ(ZIP 展開 → `wintun.dll` 配置 → `install-service.ps1`)

以降、設定ファイルは `C:\ProgramData\fips\fips.yaml`、コマンドはすべて管理者 PowerShell を想定します。

## Step 1: 永続アイデンティティ(固定 npub)を作る

既定では起動ごとに npub が変わります(エフェメラル)。自分が「到達される側」になるには固定化が必須です。

`fips.yaml` の `node:` ブロック:

```yaml
node:
  identity:
    persistent: true
```

サービス再起動すると、初回起動時に `C:\ProgramData\fips\fips.key`(秘密鍵/ nsec)と `fips.pub`(公開鍵/ npub)が生成されます。

```powershell
sc stop fips; sc start fips
fipsctl show status    # "npub": "npub1..." を控える — 以降これが自分のアドレス
Get-Content C:\ProgramData\fips\fips.pub   # 同じ npub が入っている
```

別の方法:

- 事前に鍵を生成: `fipsctl keygen -d "C:\ProgramData\fips"`(デーモン未起動でも可)
- 既存の Nostr 鍵を使う: `node.identity.nsec: "nsec1..."` を yaml に直書き(ファイルは鍵と同価なので ACL を絞ること)

> **重要**: `fips.key` / `nsec:` は秘密情報。Git にコミットしない、画面共有に写さない。`fips.pub` / npub は公開してよいもの(相手があなたを `peers:` に登録するための情報)。

## Step 2: `.fips` 名前解決を Windows 全体で使えるようにする

FIPS の DNS レスポンダはデーモン内蔵で、既定 `[::1]:5354` (UDP) で `.fips` ゾーンだけに答えます。Linux では `fips-dns-setup` が systemd-resolved へ自動登録しますが、**Windows 向けの同等機能は同梱されていません**。以下の3つの方法から選びます。

| 方法 | 仕組み | できること | 難易度 |
|------|--------|------------|--------|
| A. Windows hosts | `C:\Windows\System32\drivers\etc\hosts` に `fd..::.. name.fips` | 登録した短縮名だけ全アプリで解決 | ★簡単 |
| B. NRPT + ポート53 | NRPT で `.fips` ドメインを `127.0.0.1` に委任 + FIPS の DNS を 53番で待たせる | `<任意のnpub>.fips` が全アプリで自動解決 | ★★中 |
| C. 都度照会 | `Resolve-DnsName -Server ::1 -Port 5354` | 確認用途 | ★簡単 |

### 方法 A: hosts 登録(推奨・まずこれ)

```powershell
# npub → メッシュ IPv6 アドレスを取得(デーモン不要のローカル計算)
fipsctl address npub1qmc3cvfz0yu2hx96nq3gp55zdan2qclealn7xshgr448d3nh6lks7zel98
fipsctl address test-us01    # C:\ProgramData\fips\hosts の短縮名も使える
```

`C:\Windows\System32\drivers\etc\hosts` に追記:

```text
fd97:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx  test-us01.fips
fd97:yyyy:...                            my-friend-pc.fips
```

`ping -6 test-us01.fips`、`ssh user@test-us01.fips` などが全アプリで使えます。

> `C:\ProgramData\fips\hosts` と混同しないこと。あちらは **FIPS リゾルバー内部**の「短縮名→npub」対応表(`name.fips` が解決されるときに使われる)。Windows の hosts は **OS の名前解決**に「短縮名→IPv6 アドレス」を直接書く。

### 方法 B: NRPT ですべての `*.fips` を自動解決(高度)

Windows の NRPT(名前解決ポリシーテーブル)で `.fips` ゾーンの問い合わせを FIPS の DNS レスポンダに向けます。NRPT のネームサーバは 53番ポート固定なので、FIPS 側の待ち受けを変えます。

1. `fips.yaml` で DNS レスポンダを 53番 IPv4 ループバックに変更:

```yaml
dns:
  enabled: true
  bind_addr: "127.0.0.1"
  port: 53
```

2. 管理者 PowerShell で NRPT ルール登録:

```powershell
Add-DnsClientNrptRule -Namespace ".fips","fips" -NameServers "127.0.0.1"
Get-DnsClientNrptRule   # 登録確認
```

3. サービス再起動して検証:

```powershell
sc stop fips; sc start fips
Resolve-DnsName npub1qmc3cvfz0yu2hx96nq3gp55zdan2qclealn7xshgr448d3nh6lks7zel98.fips
ping -6 test-us02.fips
```

注意点:

- 53番を他のローカル DNS ソフト(例: Acrylic, Technitium, Docker Desktop 等)が使っていると衝突します。その場合は方法 A/C に戻す。
- `bind_addr` を `"::"` や NIC のアドレスにすると LAN/メッシュ側へ DNS を晒すことになるので、Windows では `127.0.0.1` 固定を推奨。
- 解除は `Remove-DnsClientNrptRule -Namespace ".fips","fips"` + `dns.port` を 5354 に戻す。
- これは上流が公式に同梱する Windows 向け手順ではなく、OS の標準機能(NRPT)を使う運用レシピです(2026-09 時点)。

### 方法 C: 都度照会

設定変更なしに既定のまま:

```powershell
Resolve-DnsName -Server ::1 -Port 5354 -Name <npub>.fips -Type AAAA
```

スクリプトからアドレスを取り出して使う用途向け。

## Step 3: LAN 接続(mDNS)を有効化

同一 LAN のノードを自動発見させます。

```yaml
node:
  rendezvous:
    lan:
      enabled: true
      # scope: "my-home"   # 複数の別メッシュが同居するLANでのみ設定
```

- 仕組み: `_fips._udp.local.` の DNS-SD アドバタイズを multicast し、同タイプを browse。発見した相手の UDP ポートへ直接ピア接続。
- LAN アドバタイズは未認証のヒント扱いで、実際の認証は通常どおり Noise IK ハンドシェイク。なりすましアドバタイズは握手で落ちます。
- UDP トランスポートが必要(既定で有効)。

## Step 4: グローバル接続(Nostr ディスカバリ + NAT 越え)

LAN と別系統として、インターネット越しのピア発見を有効化します。`node.identity.persistent: true` 前提(エフェメラルだと advert が再起動で無効になるため)。

### 4a. 相手を npub だけで解決する(受身側不要の最小構成)

```yaml
node:
  identity:
    persistent: true
  rendezvous:
    lan:
      enabled: true
    nostr:
      enabled: true
      advertise: false            # 自分は公開しない
      policy: configured_only

transports:
  udp:
    bind_addr: "0.0.0.0:2121"
  tcp:
    bind_addr: "0.0.0.0:8443"

peers:
  - npub: "npub1peer..."          # 相手の npub だけ知っていればよい
    alias: "friend-pc"
    via_nostr: true               # アドレスは Nostr advert から解決
    connect_policy: auto_connect
```

`via_nostr: true` のピアは `addresses:` を省略可 — 相手が公開している advert(デフォルトリレー `wss://relay.damus.io` / `wss://nos.lol` / `wss://offchain.pub` に投稿される kind 37195 イベント)からエンドポイントを取得してダイヤルします。既知の静的アドレスを併記すれば、それが優先・Nostr はフォールバックになります。

### 4b. 自分のノードを公開する(NAT 内の一般的な Windows PC)

```yaml
node:
  identity:
    persistent: true
  rendezvous:
    nostr:
      enabled: true
      advertise: true
      dm_relays:                          # 省略可(既定3リレー)。NAT越えシグナリング用
        - "wss://relay.damus.io"
        - "wss://nos.lol"
      stun_servers:
        - "stun:stun.l.google.com:19302"
        - "stun:stun.cloudflare.com:3478"

transports:
  udp:
    bind_addr: "0.0.0.0:2121"
    advertise_on_nostr: true
    public: false       # false = "udp:nat" を公開 → ホールパンチングを試みる
```

相手側は `via_nostr: true`(+ 必要なら `addresses: [{transport: udp, addr: "nat"}]`)であなたに接続します。オファー/アンサーを暗号化イベント(NIP-59 gift wrap)で交換し、両側が同時に UDP を打ち抜いて直接リンクを張ります。成功すると通常の FMP UDP トランスポートに昇格し、以降はリレー不要です。

- 片側がフルコーン/ポート制限 NAT なら高確率で成功。**対称 NAT が絡むと失敗しがち**(その場合は片側に公開ポートを用意するか、TCP 公開エンドポイントを使う)。
- ルータで UDP 2121 → この PC のポートフォワードができるなら `public: true` + `external_addr: "<グローバルIP>:2121"` で `udp:host:port` を直接公開する方が確実。

### 4c. TCP エンドポイントを公開(UDP が塞がれる相手向け)

```yaml
transports:
  tcp:
    bind_addr: "0.0.0.0:8443"
    advertise_on_nostr: true
    external_addr: "<公開IP>:8443"   # クラウド等で NIC に public IP がない場合に必須
```

### 4d. 静的ピア(テストメッシュ)を併用

Step 4a–c と併用して、`peers:` に `test-us01` を書いておくと公開テストメッシュにもつながります(Level 1 Step 4 と同じ)。

## Step 5: Windows ファイアウォール

受信を許可(サービスが接続を受ける場合に必要):

```powershell
New-NetFirewallRule -DisplayName "FIPS UDP 2121"  -Direction Inbound -Protocol UDP -LocalPort 2121 -Action Allow
New-NetFirewallRule -DisplayName "FIPS TCP 8443"  -Direction Inbound -Protocol TCP -LocalPort 8443 -Action Allow
New-NetFirewallRule -DisplayName "FIPS mDNS 5353" -Direction Inbound -Protocol UDP -LocalPort 5353 -Action Allow
```

- クライアント用途のみ(自分は公開しない)なら UDP 2121/TCP 8443 の受信許可は不要ですが、mDNS 5353 を開けないと LAN 側からの発見を受け取れません。
- `.fips` への問い合わせ(ループバック)と wintun アダプタ経由の通信は FW ルール不要。メッシュ内側(アプリから fips0 への着信)を絞りたい場合の fips.nft ファイアウォール基盤は Linux 専用で、Windows にはありません(代わりに Windows ファイアウォールの「FIPS」アダプタ向けルールや、各サービスのバインド先で制御)。

## Step 6: 検証 — 全部つながっているか

```powershell
fipsctl show status        # npub(固定済み), peer/session カウント, TUN 状態
fipsctl show peers         # LAN 発見ピア + Nostr 解決ピア + 静的ピアが connected
fipsctl show transports    # udp :2121 / tcp :8443 / (発見由来の adopt 済み udp)
fipsctl show sessions      # 確立済み FSP セッションと RTT
fipstop                    # ライブ TUI(Tab で画面切替、q で終了)
```

到達性をピア別に切り分ける(v0.5.0 新機能):

```powershell
fipsctl probe test-us01            # bloom → discovery → path → session → rtt の5段階
fipsctl probe npub1peer...        # npub 直指定も可
```

アプリから:

```powershell
ping -6 <npub>.fips                # または短縮名.fips(Step 2 の方法 A/B)
ssh user@my-friend-pc.fips         # OpenSSH(Windows同梱)
curl.exe -6 http://[fd97::...]:8080/
Test-NetConnection <npub>.fips -Port 22
```

## Step 7: 自分のサービスをメッシュに公開する

`FIPS` アダプタに割り当てられた自分の `fd97:...` アドレス(=`fipsctl address` で npub から求まるもの)にサービスをバインドするだけで、そのサービスはメッシュからのみ到達可能になります。`0.0.0.0` / `::` にバインドしたサービスにはメッシュ側からも届くので、メッシュ限定公開なら fips0 のアドレスにバインドするのが安全です。

```powershell
fipsctl address            # 自分のメッシュアドレス(fips.key から導出)
# 例: python -m http.server 8000 --bind fd97:...:<自分のアドレス>
# 相手側: curl.exe -6 http://<あなたのnpub>.fips:8000/  (名前解決は Step 2)
```

## 運用メモ

- **npub が変わった**: `persistent: true` が効いていない、または `fips.key` を読めていない(パス/権限を確認)。
- **設定を変えたら**: `sc stop fips; sc start fips`(`hosts` ファイルだけは mtime を見て自動リロードされるので再起動不要)。
- **サービス化しない運用**: `.\fips.exe -c C:\ProgramData\fips\fips.yaml` でフォアグラウンド実行も可能(管理者)。
- **アンインストール**: `uninstall-service.ps1`(設定は残る)、`-RemoveAll` で設定も削除。

## 次に進むには

各設定キーの網羅表、Nostr advert の中身、ホールパンチングの詳細、ACL(許可/拒否リスト)、トラブルシューティング集は [Level 3: 詳細リファレンス](LEVEL3-reference.md) へ。
