# Level 3: 詳細リファレンス

FIPS v0.5.0 on Windows の内部動作・設定リファレンス・セキュリティ・トラブルシューティング。上流ドキュメント(`docs/` @ v0.5.0)の Windows 視点での日本語版ダイジェスト + Windows 固有の差異まとめです。

---

## 1. Windows でのファイル・構成要素マップ

| 項目 | 場所 / 値 |
|------|-----------|
| バイナリ | `C:\Program Files\fips\fips.exe`, `fipsctl.exe`, `fipstop.exe` |
| TUN ドライバ | `wintun.dll`(`fips.exe` と同フォルダ;ZIP には**同梱されない**、別途 wintun.net から) |
| 設定ファイル | `C:\ProgramData\fips\fips.yaml`(サービスは環境変数 `FIPS_CONFIG` で特定) |
| ホスト名マップ | `C:\ProgramData\fips\hosts`(FIPS `.fips` リゾルバー用。mtime 監視で自動リロード) |
| 永続鍵 | `C:\ProgramData\fips\fips.key` (nsec) / `fips.pub` (npub)。設定ファイルの隣に生成・探索される |
| ピア ACL | `<システムドライブ>:\etc\fips\peers.allow` / `peers.deny` ※下記注意 |
| 制御ソケット | **TCP `127.0.0.1:21210`**(Unix ドメインソケットの代わり)。`fipsctl`/`fipstop` は自動接続。`-s` でポート変更 |
| TUN アダプタ | wintun アダプタ名 **`FIPS`**(アダプタ一覧に出る)。`fd…/128` を割当、`fd00::/8` のルートを netsh で追加 |
| サービス名 | `fips`(`sc start/stop/query fips`)。`fips.exe --install-service` / `--uninstall-service` |
| ログ | **stdout のみ**(ファイル/Event Log 出力なし)。サービスでは実質見えない → デバッグはフォアグラウンド `.\fips.exe -c <conf>` で `node.log_level: debug` または `RUST_LOG` |

> **ACL パスの注意(上流の現状)**: `hosts` は `C:\ProgramData\fips\hosts` に Windows 用パスが定義されていますが、`peers.allow`/`peers.deny` の既定パスは Windows でも `/etc/fips/` 系のままです(ソース上 macOS/FreeBSD 以外は `/etc/fips/`)。つまり Windows では `C:\etc\fips\peers.allow` を読みに行きます。ACL を使う場合はこの場所に配置する必要があります(v0.5.0 時点の挙動。今後修正される可能性あり)。

## 2. Windows で使える/使えない機能

| 機能 | Windows |
|------|---------|
| UDP / TCP トランスポート | ✅ |
| Tor トランスポート | ✅(tor の SOCKS5 が別途必要) |
| Nym(mixnet) | ✅(nym-socks5-client が別途必要) |
| mDNS LAN 発見 (`rendezvous.lan`) | ✅ |
| Nostr リレー仲介ディスカバリ / NAT 越え | ✅ |
| `.fips` DNS レスポンダ(デーモン内蔵) | ✅ `[::1]:5354` |
| `.fips` の OS リゾルバ統合 | ❌ 同梱なし → hosts / NRPT で自前構成(Level 2 Step 2) |
| Ethernet(生 L2)トランスポート | ❌ |
| BLE トランスポート | ❌ |
| `fips-gateway`(LAN ゲートウェイ) | ❌(nftables が Linux 専用) |
| ネイティブ datagram API | ❌(Unix のみ) |
| メッシュ I/F ファイアウォール(fips.nft) | ❌(Linux のみ) |
| systemd 相当 | Windows サービス(SCM)。`--install-service` で登録 |

> **「Ethernet ❌」は「LAN 非対応」ではありません。** ここで言う Ethernet トランスポートは *IP を介さない生レイヤー2フレーム*で直接メッシュを張る ground-up モード用のもので、Windows では未実装です。通常の LAN(IP ネットワーク)内での通信は **UDP トランスポート + mDNS 発見**(`rendezvous.lan`)で問題なく動きます。

## 3. `fips.yaml` 設定リファレンス(Windows 向け主要項目)

完全なテンプレ例:

```yaml
node:
  identity:
    persistent: true            # 固定 npub。または nsec: "nsec1..."
  # log_level: "info"           # trace|debug|info|warn|error。RUST_LOG が優先
  # drain_timeout_secs: 2       # シャットダウン時のドレイン猶予
  control:
    enabled: true
    socket_path: 21210          # Windows では TCP ポート番号

  rendezvous:
    lan:                        # mDNS LAN 発見(既定 off)
      enabled: true
      # scope: "my-net"         # 同居する別メッシュと分離したい時だけ
      # service_type: "_fips._udp.local."
    nostr:                      # Nostr リレー仲介(既定 off)
      enabled: true
      advertise: true           # 自分のエンドポイントを advert 公開
      policy: configured_only   # disabled | configured_only | open
      # app: "fips-overlay-v1"  # discovery 名前空間(open 時は変えると良い)
      advert_relays:            # advert 公開先(既定は下記3本)
        - "wss://relay.damus.io"
        - "wss://nos.lol"
        - "wss://offchain.pub"
      dm_relays:                # NAT越えシグナリング(NIP-59)用リレー
        - "wss://relay.damus.io"
        - "wss://nos.lol"
      stun_servers:
        - "stun:stun.l.google.com:19302"
        - "stun:stun.cloudflare.com:3478"
        - "stun:global.stun.twilio.com:3478"
      # open_discovery_max_pending: 64
      # advert_ttl_secs: 3600 / advert_refresh_secs: 1800
      # punch_start_delay_ms: 2000 / punch_interval_ms: 200 / punch_duration_ms: 10000
      # share_local_candidates: false   # 同一LANのRFC1918候補も載せる(通常 off)

tun:
  enabled: true                 # wintun「FIPS」アダプタ。要管理者
  name: fips0                   # Windows ではアダプタ表示名は "FIPS" 固定
  mtu: 1280

dns:
  enabled: true
  bind_addr: "::1"              # NRPT 構成時は "127.0.0.1" に
  port: 5354                    # NRPT 構成時は 53 に
  ttl: 300

transports:
  udp:
    bind_addr: "0.0.0.0:2121"
    mtu: 1280
    # advertise_on_nostr: true  # advert に UDP を載せる
    # public: false             # true=公開アドレス直接 / false=udp:nat(ホールパンチ)
    # external_addr: "x.x.x.x:2121"  # public:true 時の公開アドレス固定
    # outbound_only: false      # true = 純クライアント(受信不可・advert 不可)
    # accept_connections: true  # false で新規 inbound 握手を拒否
  tcp:
    bind_addr: "0.0.0.0:8443"
    # advertise_on_nostr: true
    # external_addr: "x.x.x.x:8443"
  # tor:
  #   mode: directory           # onion サービス公開側
  #   socks5_addr: "127.0.0.1:9050"
  #   advertised_port: 8443
  #   directory_service: { hostname_file: ..., bind_addr: "127.0.0.1:8444" }
  #   advertise_on_nostr: true

peers:
  - npub: "npub1..."            # 必須。相手の Nostr 公開鍵
    alias: "name"               # 任意。ログ/fipsctl の表示名 + name.fips 解決
    addresses:                  # via_nostr だけにするなら省略可
      - transport: udp          # udp|tcp|tor|nym  (+ "/インスタンス名" 修飾も可)
        addr: "host:port"       # または "nat"(ホールパンチ指定)
    via_nostr: false            # true で advert 由来エンドポイントを追加
    connect_policy: auto_connect  # 実質これのみ有効(on_demand/manual は将来用)
```

- 旧来の `node.discovery.*` キーは v0.5.0 で `node.rendezvous.*` に改名。旧スペルも読めるが deprecation 警告が出る。
- テストメッシュの短縮名→npub 一覧は `C:\ProgramData\fips\hosts` に同梱済み(`test-us01` … `test-uk01`)。

## 4. `<npub>.fips` 名前解決の内部動作

1. アプリが `<npub>.fips` を引く → OS リゾルバ → (NRPT/hosts の設定次第で) FIPS DNS レスポンダへ。
2. レスポンダの解決順序: **① `hosts` ファイル(短縮名→npub)→ ② `peers[].alias` → ③ 名前自体が有効な npub なら直接変換 → ④ NXDOMAIN**。
3. npub → `fd00::/8` アドレスは**決定論的ハッシュ**(登録・照会不要)。`fipsctl address` がこの計算をオフラインで行う。
4. レスポンダは `.fips` ゾーン専用のスタブ(再帰しない)。`bind_addr: "::"` でメッシュ越し公開も可能だが、mesh I/F 経由クエリはフィルタされる設計(Windows では ifindex 取得が未対応でフィルタ無効 — `::` 公開は避けるべき)。
5. Windows での実用構成は Level 2 Step 2。**NRPT は「ドメイン別に使う DNS サーバを指定する」OS 標準機能**で、`.fips` → `127.0.0.1`(FIPS を `127.0.0.1:53` で待たせる)が全アプリ透過の最も近い手段。

## 5. グローバル接続の内部動作(Nostr ディスカバリ + NAT 越え)

- **advert**: kind **37195** の replaceable イベントに「受付可能なトランスポート/エンドポイント/リレー/TTL」を署名して advert_relays に投稿。既定 TTL 3600 秒、30 分ごとに再投稿。
- **ピア解決**: `via_nostr: true` のピアは advert_relays を購読してエンドポイントを得る。`policy: configured_only` は設定済みピアのみ、`open` は同一 `app` 名前空間の advert をすべて接続候補にする無承認モード。
- **NAT 越え (`udp:nat`)**: `public: false` で `udp:nat` エンドポイントを公開。接続時に NIP-59 gift wrap で暗号化オファーをピア宛に送り、アンサーを受けた両側が `punch_start_delay_ms`(既定2秒)後に `punch_interval_ms`(200ms)間隔で同時 UDP 送信開始、`punch_duration_ms`(10秒)で打ち切り。成立したソケットは FMP UDP トランスポートとして adopt → Noise IK → 以降リレー不要。
- **限界**: 対称 NAT( symmetric NAT )が片側でもあると大体失敗。打ち抜き以外の代替はプロトコル内にない → その場合は片側に公開 UDP/TCP エンドポイント(ポートフォワード or クラウド側)を用意。`share_local_candidates: true` は「本当に同一 LAN の相手」限定(RFC1918 候補を advert に載せる)。VPN/Tailscale 等で誤判定する環境では off のままに。
- **リレーはディスカバリ面のみ**: advert/シグナリングに使うだけで、メッシュのデータは流れない(Tor の場合もデータ面だけが Tor)。
- セキュリティモデルの詳細: advert の改ざんは署名で検出されるが、リレーに流量が見える点は `docs/design/fips-nostr-discovery.md` 参照。

## 6. LAN(mDNS)の内部動作

- `_fips._udp.local.` サービスを mDNS で publish + browse。TXT に npub(と任意 `scope`)。
- アドバタイズは**未認証のヒント** — 身元は必ず接続後の Noise IK で検証されるため、偽のアドバタイズは握手で弾かれる。
- `scope` を設定したノード同士だけが互いを見る。既定は無スコープ(同一 LAN の全 FIPS ノードが候補)。
- Windows では「ネットワーク プロファイル」が「パブリック」だと mDNS の受信が FW に弾かれやすい — UDP 5353 受信許可(Level 2 Step 5)かネットワークを「プライベート」に。

## 7. セキュリティ要点

### 「運ぶノード」と「読めるノード」は分離している

- **リンク層(FMP / Noise IK)**: 直接ピア間だけの hop-by-hop 暗号化。中継ノードはリンク層だけを復号して次ホップへ転送する。
- **セッション層(FSP / Noise XK)**: 宛先 npub とのエンドツーエンド暗号。中継ノードはペイロードを読めない。見えるのは宛先ノードアドレス(公開鍵のハッシュ)程度。
- ピアを許可する = 「自分のトラフィックの経路に入りうる相手」を決めること。中身を読まれることにはならないが、転送を請け負う/負わせる関係になる。

### ピアリングの許可制御(admission)

- 既定は **default-allow**: トランスポートに届いて Noise IK を完遂したノードは誰でもピアになれる。
- 制御手段: `peers.allow`/`peers.deny`(allow 一致→許可、deny 一致→拒否、未一致→許可;厳格許可制は allow 列挙 + deny に `ALL`)、`accept_connections: false` / `outbound_only: true`、Nostr の `policy`(`configured_only` か `open`。open は無承認 → ACL 必須)。
- Windows の ACL パスは §1 の注意参照(`C:\etc\fips\`)。

### その他

- **制御ソケット(TCP 127.0.0.1:21210)に ACL なし**: ローカルの任意プロセスが `connect`/`disconnect`/`inject-config` 可能。単独ユーザー端末向け。共用機では FW で 21210 への接続を絞ること。
- **`fips.key` は nsec** — 平文を Git/チャットに出さない。Windows は親フォルダ ACL 継承(Unix の 0600 相当の強制はない)。`C:\ProgramData\fips` は既定で管理者のみ書き込み可。
- **`policy: open` は無承認**: ACL を正しく設定してから使う。
- **メッシュ内側のサービス露出**: fips0 宛に来るトラフィックはホスト FW 次第(Windows に fips.nft 相当はない)。メッシュ限定サービスは fips0 のアドレスにバインド。
- `node.identity.nsec:` を yaml に書く方式は、設定ファイル=秘密情報として ACL を厳しく。

## 8. トラブルシューティング

| 症状 | 確認・対処 |
|------|-----------|
| サービスが `sc start` で即落ち | フォアグラウンド `.\fips.exe -c C:\ProgramData\fips\fips.yaml` を管理者で実行し stdout のエラーを見る。`node.log_level: debug` か `RUST_LOG=debug` |
| 「Failed to load wintun.dll」 | `wintun.dll` が `fips.exe` と同フォルダにあるか。amd64 版か。管理者実行か |
| `fipsctl` がつながらない | `fipsctl -s 21210` / `sc query fips` でデーモン生存確認。TCP 21210 が listen しているか `netstat -ano | findstr 21210` |
| ピアが connected にならない | `fipsctl show connections` で握手途中のものを見る → 外向 UDP 遮断なら TCP 443 ピアへ。時計ずれ(署名検証失敗)も原因になる |
| `via_nostr` で解決しない | 相手が `advertise: true` で advert を出しているか / 自分の `rendezvous.nostr.enabled: true` / リレーへの WSS 接続が通るか(企業 NW は 443/WSS 遮断に注意) |
| `udp:nat` がつながらない | 対称 NAT の可能性。`punch_duration_ms` を伸ばすか、片側を `public: true` + ポートフォワード or TCP 公開に |
| LAN で発見されない | `rendezvous.lan.enabled` 両側で on / UDP 5353 受信許可 / ネットワークプロファイル「プライベート」/ 同一 L2 セグメントか(VLAN・AP isolation に注意) |
| `<npub>.fips` が引けない | `Resolve-DnsName -Server ::1 -Port 5354 -Name ...` でレスポンダ直接確認 → 応答するなら OS 統合側(hosts/NRPT)の問題。NRPT 時は FIPS が `127.0.0.1:53` で listen しているか `netstat` 確認 |
| ping は通るが TCP が落ちる | MTU 問題の可能性。`fipsctl show sessions` の `path_mtu`、上流の diagnose-mtu ハウツー参照。`tun.mtu` を 1280→それ以下で試す |
| `test-us02` 等トランジット先に届かない | テストメッシュ側の一時的経路不良のことが多い — 少し待つ/別ノードを試す。自ピアへの ping が通っていれば自側は健全 |
| アドレスだけ知りたい | `fipsctl address [npub|短縮名]`(デーモン不要) |
| 5段階の切り分け | `fipsctl probe <対象>`: bloom(メッシュがその宛先を知っているか) → discovery → path → session → rtt |

## 9. 用語・プロトコル対応表

| 用語 | 意味 |
|------|------|
| npub / nsec | Nostr の公開鍵/秘密鍵(bech32)。FIPS のノード ID |
| FMP | メッシュ層プロトコル。ピア間 hop-by-hop、Noise IK 暗号化 |
| FSP | セッション層。エンドツーエンド Noise XK。ポート 256 = IPv6 shim |
| spanning tree + bloom | 座標ベースのルーティング + 到達可能性広告(1KB 固定フィルタ) |
| MMP | リンク別 RTT/loss/jitter/goodput 計測プロトコル |
| rendezvous | ピア発見の総称。`lan`(mDNS)と `nostr`(リレー+STUN) |
| advert | kind 37195 Nostr イベント。エンドポイント公開 |
| `udp:nat` | NAT 内ノードの advert エンドポイント。ホールパンチ合図 |
| fips0 / "FIPS" | TUN アダプタ(Windows では wintun、アダプタ名 FIPS) |

## 10. 参照リンク

- 上流リポジトリ: <https://github.com/jmcorgan/fips> (`v0.5.0` タグ)
- リリースノート v0.5.0: `docs/releases/release-notes-v0.5.0.md`
- マルチプラットフォーム導入: `docs/getting-started.md`
- チュートリアル: `docs/tutorials/`(join-the-test-mesh / persistent-identity / resolve-peers-via-nostr / advertise-your-node / open-discovery / reach-mesh-services / host-a-service / ground-up-mesh)
- ハウツー: `docs/how-to/`(enable-nostr-discovery / host-aliases / enable-mesh-firewall(Linux) / deploy-tor-onion / diagnose-mtu-issues 等)
- リファレンス: `docs/reference/configuration.md`(全設定キー)、`cli-fipsctl.md`、`nostr-events.md`、`security.md`
- 設計: `docs/design/`(fips-concepts → fips-architecture → fips-ipv6-adapter → fips-nostr-discovery)
- wintun: <https://www.wintun.net/>
