# Level 1: 最小構成(クイックスタート)

**目的**: FIPS v0.5.0 を Windows にインストールし、公開テストメッシュに参加して `<npub>.fips` 宛の疎通まで確認する最短ルート。所要約15分。

> このレベルでは「接続する側(クライアント)」として動かします。自分のノードを外部から到達可能にする設定や、全アプリでの `.fips` 名前解決は Level 2 で扱います。

---

## Step 1: ファイルを入手する

1. [FIPS v0.5.0 リリースページ](https://github.com/jmcorgan/fips/releases/tag/v0.5.0) から **`fips-0.5.0-windows-x86_64.zip`** と **`checksums-windows.txt`** をダウンロード。
2. [wintun.net](https://www.wintun.net/) から `wintun-x.x.x.zip` をダウンロードし、中の **`wintun/bin/amd64/wintun.dll`** を取り出す。

チェックサム検証(任意だが推奨):

```powershell
Get-FileHash .\fips-0.5.0-windows-x86_64.zip -Algorithm SHA256
# checksums-windows.txt の該当行と一致することを確認
```

## Step 2: 展開と配置

```powershell
# 管理者 PowerShell で
Expand-Archive .\fips-0.5.0-windows-x86_64.zip -DestinationPath C:\fips
# wintun.dll を fips.exe と同じフォルダに置く
Copy-Item .\wintun.dll C:\fips\
```

ZIP の中身: `fips.exe` / `fipsctl.exe` / `fipstop.exe` / `fips.yaml` / `hosts` / `install-service.ps1` / `uninstall-service.ps1`。

## Step 3: Windows サービスとしてインストール

```powershell
# 管理者 PowerShell で
cd C:\fips
.\install-service.ps1
```

このスクリプトが行うこと:

- `fips.exe` / `fipsctl.exe` / `fipstop.exe` / `wintun.dll` → `C:\Program Files\fips\`
- 設定ファイル → `C:\ProgramData\fips\fips.yaml`、ホスト名マップ → `C:\ProgramData\fips\hosts`
- マシン環境変数 `FIPS_CONFIG=C:\ProgramData\fips\fips.yaml` を設定
- `C:\Program Files\fips` をシステム PATH に追加
- `fips` という名前の Windows サービスを登録

> 手動で試したい場合はサービス化せず `.\fips.exe -c fips.yaml` でフォアグラウンド起動も可能(管理者必須)。設定ファイルは exe と同じフォルダ、`%APPDATA%\fips\`、または環境変数 `FIPS_CONFIG` の順で探される。

## Step 4: グローバル接続 — テストメッシュに参加

`C:\ProgramData\fips\fips.yaml` を管理者権限のエディタで開き、`peers: []` の行を以下に置き換える:

```yaml
peers:
  - npub: "npub1qmc3cvfz0yu2hx96nq3gp55zdan2qclealn7xshgr448d3nh6lks7zel98"
    alias: "test-us01"
    addresses:
      - transport: udp
        addr: "test-us01.fips.network:2121"
    connect_policy: auto_connect
```

サービスを起動(または再起動):

```powershell
sc start fips        # 初回
sc stop fips; sc start fips   # 設定を変えた後は再起動
```

確認:

```powershell
fipsctl show status      # 自分の npub とメッシュ IPv6 アドレスが見える
fipsctl show peers       # test-us01 が connectivity: connected になるのを待つ
fipsctl show transports  # UDP :2121 / TCP :8443 リスナー
```

接続に成功すると `show peers` で `display_name: test-us01`、`connectivity: connected` になります。30秒経っても `connected` にならない場合は Level 3 のトラブルシューティングへ(外向き UDP が塞がれている環境では TCP:443 に切り替える)。

## Step 5: `<npub>.fips` で疎通確認

FIPS の DNS リゾルバーはデーモン内にあり、既定で `[::1]:5354` で待ち受けます。Windows には `.fips` をシステムリゾルバーへ自動登録する仕組みが同梱されていないため、Level 1 では次のどちらかで確認します。

**方法 A — DNS 直接問い合わせ + IPv6 アドレスで ping:**

```powershell
# .fips 名 → メッシュ IPv6 アドレス
Resolve-DnsName -Server ::1 -Port 5354 `
  -Name npub1qmc3cvfz0yu2hx96nq3gp55zdan2qclealn7xshgr448d3nh6lks7zel98.fips `
  -Type AAAA

# 戻ってきた fd97:... のアドレスへ ping(自ノードの fips0 経由)
ping -6 fd97:...:<先ほどのアドレス>
```

**方法 B — hosts に短縮名を登録して `ping <名前>.fips`:**

インストールで `C:\ProgramData\fips\hosts` にテストメッシュの名前一覧が既に入っています。Windows 本体の hosts(`C:\Windows\System32\drivers\etc\hosts`)に、メッシュアドレスと `.fips` 名を1行書くと全アプリが使えます:

```powershell
fipsctl address test-us01   # または npub を直接渡す → fd97:... が返る
# C:\Windows\System32\drivers\etc\hosts に追記(管理者):
#   fd97:...:<addr>    test-us01.fips
ping -6 test-us01.fips
```

直接ピア(`test-us01`)だけでなく、設定していない `test-us02` なども ping できるはずです — 1つの良いピアがあればメッシュ経由で他ノードへ届くのが FIPS の核心です。

## Step 6: LAN(ローカル)接続 — mDNS 自動発見

同じ LAN に別の FIPS ノードを立てる場合、`fips.yaml` の `node.rendezvous` 配下を有効化します:

```yaml
node:
  rendezvous:
    lan:
      enabled: true
```

`sc stop fips; sc start fips` で再起動。あとは自動です — 同一 LAN 上のノードが `_fips._udp.local.` の mDNS アドバタイズを見つけ、`fipsctl show peers` に数秒で現れます。Windows 側では UDP 5353(mDNS)と UDP 2121(FIPS)の受信を Windows ファイアウォールで許可してください(コマンドは Level 2 Step 6)。

---

## ここまででできたこと

- FIPS デーモンが Windows サービスとして動き、`FIPS` という wintun アダプタに `fdxx:...` のメッシュ IPv6 が振られている
- `test-us01` 経由で公開テストメッシュに参加し、任意ノードへ `<npub>.fips` / `fd97:...` で通信できる
- LAN 内ノードは mDNS で自動発見される

## 次に進むには

- **永続 npub**(再起動しても ID が変わらない)と **自分のノードを公開する**(Nostr advert / NAT 越え)、**全アプリでの `*.fips` 自動解決** → [Level 2: 標準構成](LEVEL2-standard.md)
- 設定項目の網羅・内部動作・セキュリティ → [Level 3: 詳細リファレンス](LEVEL3-reference.md)
