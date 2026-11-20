# research

braindump.md の主張・事例の裏取り結果。出典つき。
最終更新: 2026-11-17(一次情報の確認と未調査事例の調査を反映)

凡例:
- 🟢 一次情報(当事者の報告・公式ドキュメント・論文本文・公式統計)を直接読んで確認
- ✅ 一次は読めていないが、複数の二次情報で一致
- ⚠️ 食い違い・一部のみ確認・解釈を含む
- ❌ 確認できなかった / 誤りだった

---

## 早見表: 事例ごとに「どの対策で止まったか」

| 事例 | 時期 | 実行のタイミング・経路 | install スクリプトのブロック | Provenance | 備考 |
|---|---|---|---|---|---|
| PhantomRaven | 2025-08〜 | HTTP URL の依存(RDD)を取得し、その中の preinstall で実行 | 🟢 止まる(preinstall) | — | **検知ツールをすり抜けた**(依存が「0」に見える)。npm v12 の `--allow-remote` 既定 none でも止まる |
| chalk / debug ほか18パッケージ | 2025-09-08 | **ブラウザ上の実行時**に fetch / XHR / ウォレット API をフック | 🟢 効かない | — | install 時の対策は無関係 |
| node-ipc | 2026-05-14 | **`require()` 時** | 🟢 効かない | — | 休眠メンテナの期限切れドメイン経由の乗っ取りの可能性 |
| Phantom Gyp(Miasma) | 2026-06-03 | `binding.gyp` による**暗黙の node-gyp rebuild** | 🟢 止まる(`--ignore-scripts`、pnpm のビルド承認) | Sigstore 証明を偽造 | **`scripts` を見るチェックは素通り** |
| AsyncAPI | 2026-07-14 | **`require()` 時** | 🟢 効かない | 🟢 **本物の Provenance 付き** | 正規の CI が公開 |
| keyv / cacheable(ChainDrop) | 2026-08-04 | npm 側は **preinstall**。リポジトリ側は **IDE / AI エージェントのフック**(開くだけで実行) | 🟢 npm 側は止まる / リポジトリ側は無関係 | 🟢 **本物の Provenance 付き** | トークン失効で発火する仕掛け |

読み取れること(⚠️ ここは解釈):
- **「install スクリプトのブロック」で止まる攻撃と止まらない攻撃が、同じ時期に並行して存在している**(置き換わりではなく多様化)
- 止まった事例でも、検知(PhantomRaven)や Provenance(keyv)は別の場所で破られている
- Phantom Gyp・chalk・keyv はいずれも数時間で公開〜発覚〜削除まで進んでおり、**1日の Minimum Release Age は新規インストールには効いた可能性が高い**。ただし `require()` 時に動く型でも同じで、Minimum Release Age は「実行のタイミング」ではなく「公開からの経過時間」で守る別軸の対策

---

## node-ipc(import 時に発火)

- 🟢 2026-05-14(約14:25 UTC)に悪意ある3バージョン(9.1.6 / 9.2.3 / 12.0.1)が公開。直前の正規リリースは 2026-05-07(Snyk)
- 🟢 ペイロードは CommonJS エントリポイントに即時実行関数として追加され、**import 時に実行**。preinstall / install / postinstall には依存しない(Snyk)
- 🟢 90 超の認証情報カテゴリを狙い、`azurestaticprovider[.]net` へ送信(Snyk が StepSecurity の報告として引用)
- 🟢 経路は、**期限切れになったメンテナのメールドメインを再取得し、npm のアカウント復旧を悪用した**可能性。ただし Snyk は「完全な証拠ではなく手がかり」としている。レジストリ・ソースリポジトリ・CI は侵害されていない可能性が高い
- 🟢 2022年3月の peacenotwar(プロテストウェア)の件とは**別の事件**で、今回は認証情報窃取(Snyk)
- 🟢 **週間ダウンロード数: 854,545**(2026-05-07〜05-13、攻撃直前の週。npm 公式ダウンロード API)→ 「週 約85万」で確定。「1,000万超」は誤り
- ❌ 「Socket が約3分で検知」: Socket の原文はアクセス拒否(403)で読めず、他のソースにもこの数値はなかった。**使わない**

出典:
- Snyk: https://snyk.io/blog/malicious-node-ipc-versions-published-npm
- npm ダウンロード API: https://api.npmjs.org/downloads/point/2026-05-07:2026-05-13/node-ipc
- Socket(未読・403): https://socket.dev/blog/node-ipc-package-compromised

## Phantom Gyp(binding.gyp 経由の暗黙ビルド)

- 🟢 **命名は StepSecurity**(原文「a technique we are calling "Phantom Gyp"」)。キャンペーン名は Miasma。JFrog 命名説は二次情報の誤りの可能性が高い
- 🟢 2026-06-03。57 パッケージ・286 超のバージョン。2時間未満のローリングキャンペーン(StepSecurity)
- 🟢 `package.json` に install スクリプトの宣言は一切なく、157 バイトの `binding.gyp` を追加。npm は `binding.gyp` を見つけると自動で `node-gyp rebuild` を実行する。gyp のコマンド置換 `<!(...)` で `node index.js` を実行(StepSecurity)
- 🟢 StepSecurity は「セキュリティツールが通常監視するライフサイクルスクリプトの代わり」「**ほとんどの install-script セキュリティチェックを完全にバイパス**」と表現 = `scripts` フィールドを見るチェックが素通りされた
- 🟢 **`--ignore-scripts` は npm の暗黙の node-gyp rebuild もブロックする**(Snyk)。ただし「唯一の防御と見なすべきではない」: パッケージ自体は取得・展開される、後続の `npm rebuild` などでビルドされ得る、他のパッケージマネージャーでは挙動が違い得る
- ⚠️ StepSecurity の対策欄は「`--ignore-scripts` は postinstall をブロックし、自動 node-gyp ビルドをブロックすることで binding.gyp 経路が閉じる」と、**2つが別の対策に読める書き方**。「ignore-scripts では gyp が止まらない」という印象の出所になり得る(推測)
- 🟢 ワームは盗んだ npm トークンでメンテナの全パッケージを列挙し、`binding.gyp` と `index.js` を注入して再公開。**Sigstore の Provenance 証明も偽造**する。RubyGems では `extconf.rb` で同種の手口。GitHub リポジトリには `.claude/` などの AI エージェント / エディタのフックも仕込む(Snyk)
- 🟢 窃取情報の持ち出し先に GitHub を使う(CI では `github.com` への通信は遮断されにくいため)(Snyk)

### 手元検証: `--ignore-scripts` は暗黙の gyp ビルドを止めるか(2026-11-15)

scripts なし + `binding.gyp` のみのパッケージ(gyp の `<!(...)` でマーカーファイルを作る)を tarball でインストールした。

| 環境 | 暗黙の gyp ビルド |
|---|---|
| npm 10.9.9(デフォルト) | 実行された |
| npm 10.9.9 + `--ignore-scripts` | **実行されなかった** |
| pnpm 10.28.0(デフォルト) | **実行されなかった**(ビルド承認待ちの警告) |

→ Snyk の記述と一致。限界: 単純な1パターンのみ。yarn / bun は未検証

出典:
- StepSecurity: https://www.stepsecurity.io/blog/binding-gyp-npm-supply-chain-attack-spreads-like-worm
- Snyk: https://snyk.io/blog/node-gyp-supply-chain-compromise-self-propagating-npm-worm-binding-gyp

## npm v12

- 🟢 **2026-06-09 に GitHub が v12 の破壊的変更を予告**。この予告の時点で「`binding.gyp` があり install スクリプトがないパッケージも、npm が暗黙の `node-gyp rebuild` を走らせるのでブロックされる」と明記されている
  - → **Phantom Gyp(06-03)の6日後の予告に、すでに gyp のブロックが含まれていた**。ただし、Phantom Gyp を受けて入れた変更なのかは原文に書かれていない。⚠️「Phantom Gyp に対応して塞いだ」とは言い切らず、「直後に予告された v12 で塞がれた」という時系列に留める
  - 予告時点で、これらの変更は npm 11.16.0 以降で警告として利用可能だった
- 🟢 **2026-07-08 に v12 が GA(latest タグ)**(GitHub Changelog)
- 🟢 v12 で既定になったもの:
  - `allowScripts` 既定オフ: 依存の preinstall / install / postinstall と**暗黙の node-gyp ビルド**は、明示的に許可しない限り動かない
  - `--allow-git` 既定 none: Git 依存(直接・推移的)を解決しない
  - `--allow-remote` 既定 none: リモート URL(https の tarball など)の依存を解決しない → **PhantomRaven の RDD 経路も閉じる**
  - `npm approve-scripts` / `npm deny-scripts` で許可リストを管理し、`package.json` にコミットする
- 🟢 npm 公式ドキュメント: `ignore-scripts`(既定 false)「true なら package.json に書かれたスクリプトを実行しない」。`min-release-age`(既定 null、日数)もある。満たすバージョンがなければエラーになる
- 🟢 同じリリースで、2FA バイパスの granular access token の、最も機微な用途の廃止も始まった

出典:
- GitHub Changelog(06-09 予告): https://github.blog/changelog/2026-06-09-upcoming-breaking-changes-for-npm-v12/
- GitHub Changelog(07-08 GA): https://github.blog/changelog/2026-07-08-npm-install-time-security-and-gat-bypass2fa-deprecation/
- npm docs(config): https://docs.npmjs.com/cli/v12/using-npm/config

## AsyncAPI(本物の Provenance 付きで侵害)

- 🟢 2026-07-14 07:10 UTC。2つの AsyncAPI リポジトリから、4パッケージの悪意あるバージョンが公開された(StepSecurity)
  - generator 3.3.1 / generator-helpers / generator-components、specs 6.11.2 と 6.11.2-alpha.1 など(StepSecurity のリストに 6.11.1 も並んでいるが、直前の正規版との対比の可能性あり。⚠️ バージョンを画面に出すなら原文の表を再確認)
- 🟢 攻撃者は next ブランチへの push 権限を得て、プロジェクト自身の GitHub Actions リリースワークフロー + npm の OIDC Trusted Publisher で公開させた。**StepSecurity は「npm トークンは盗まれていない」と明言**
- 🟢 原文: 「パッケージは正当な SLSA Provenance 証明を持つ。証明するのはプロジェクトの正規ワークフローが作ったことだけで、**それを起動したコミットが正当だったことではない**。Provenance は侵害された push 認証情報からは守らない」
- 🟢 ドロッパーは**ライブラリの読み込み時(`require()`)に実行**され、インストール時ではない
- 🟢 同日 11:18 UTC までに5バージョンすべてが unpublish。「新規インストールは安全、露出期間中の既存インストールと lockfile は安全ではない」
- 🟢 **Microsoft の原文(WebFetch で取得)で補強**:
  - 影響は「4パッケージ名・5バージョン」: @asyncapi/specs 6.11.2-alpha.1 / 6.11.2、@asyncapi/generator 3.3.1、@asyncapi/generator-components 0.7.1、@asyncapi/generator-helpers 1.1.1。**安全版は specs 6.11.1 以前**(→ StepSecurity の表の 6.11.1 は直前の正規版。⚠️ は解消)
  - @asyncapi/specs は多数の AsyncAPI ツールの**推移的依存**なので、直接使っていなくても影響が及んだ
  - 発端は GitHub Actions の **「pwn request」**: PR #2155 が `pull_request_target` のワークフローで信頼できない head commit をチェックアウトさせ、**asyncapi-bot の PAT が露出**した。ただし「認証情報がどう取得されたかは確定していない」と明記
  - その後 asyncapi-bot として自動公開ブランチに push し、正規の release ワークフローに公開させた。「npm を侵害したり、別の公開経路を作ったりする必要はなかった」
  - 「5バージョンすべてが GitHub OIDC による npm の trusted publishing で公開され、**有効な provenance 証明を持っていた**」「証明は、パッケージを作った正規のリポジトリ・コミット・ワークフローを**正確に**示していた。**起動したコミットが無許可だったにもかかわらず**」
  - 「よくある postinstall フック型と違い、このキャンペーンは**モジュール読み込み(import / require)時**に実行される」「影響を受けたパッケージはどれも preinstall / install / postinstall を宣言していなかった」「**緩和策として `npm install --ignore-scripts` に頼らないこと**」
  - 推奨策には「npm CLI 11.10.0 以上への更新、または `min-release-age` の利用」が含まれる。公開する側への推奨として、トークンのスコープ、ワークフローの承認、protected environment、公開の異常検知の見直し
- ⚠️ Aikido は「npm トークンが盗まれた」としている(二次情報経由)。StepSecurity は「npm トークンは盗まれていない」、Microsoft は「npm を侵害する必要はなかった」で、**一次情報2つは「npm トークン窃取ではなく、リポジトリ側の権限(bot の PAT)経由」で一致**

出典:
- StepSecurity: https://www.stepsecurity.io/blog/compromised-next-branch-pushes-malicious-asyncapi-generator-generator-helpers-and-generator-components-to-npm
- Microsoft: https://www.microsoft.com/en-us/security/blog/2026/07/15/unpacking-asyncapi-npm-supply-chain-compromise-import-time-payload-delivery/

## keyv / cacheable(ChainDrop)

- 🟢 2026-08-04。keyv のメンテナの **GitHub アカウント**が乗っ取られた(Wiz)。9:00 UTC から、まずリポジトリに IDE の永続化ペイロードを仕込み、その後に悪意あるバージョンを公開(Wiz)
- 🟢 npm 側の経路は **`"preinstall": "node setup.mjs"`**。Bun ランタイムを取得して難読化された第2段を実行し、クラウド鍵・Vault・Kubernetes・GitHub Actions のシークレット・npm トークンなどを収集。盗んだ npm トークンで他のパッケージにも同じフックを注入し、再公開する**ワーム**(SANS ISC)
  - → **install スクリプトをブロックしていれば npm 経由の経路は止まる**。流行の型も、まだ現役
- 🟢 リポジトリ側の経路: `.claude/settings.json` の SessionStart と `.vscode/tasks.json` の folderOpen。**クローンしたフォルダを開くだけで実行**され、インストールもビルドも不要(SANS ISC、StepSecurity)
- 🟢 **本物の SLSA Provenance 付き**。メンテナの GitHub アカウントで汚染したコミットとタグを push し、正規のリリースワークフローに公開させた。StepSecurity の原文: 「Provenance はどのコミットがビルドされたかを証明する。そのコミットが承認されたものだとは証明できない」
- 🟢 規模: **444 パッケージ・2,212 バージョン、4時間未満**(StepSecurity。09:40〜13:20 UTC)。SANS は「440 超・2,000 超のバージョン」。食い違いはなく、「11リリース」は初期の数字と思われる
- 🟢 名前: StepSecurity は **ChainDrop** と呼んでいる。❌ 壁打ちメモの「Wave Six」はどのソースにも出てこない → 使わない
- 🟢 **トークン失効で発火する仕掛け**(SANS ISC「Don't Revoke That Token Yet」):
  - 盗んだ GitHub トークンを `~/.config/gh-token-monitor/` に書き込み、macOS の LaunchAgent / Linux の systemd ユーザーサービスとして常駐(説明は「GitHub Token Validity Monitor」と無害を装う)
  - 60秒ごとにそのトークンで GitHub API を叩き、**トークンが失効して 4xx が返った瞬間に、遠隔から渡されたハンドラ文字列を eval**し、痕跡を消して終了する
  - ハンドラの中身は実行時に取得されるため、公開分析では不明(データ破壊かもしれないし、何もしないかもしれない)
  - 推奨手順: **先にネットワークから隔離**(4xx が返らないので発火しない)→ 常駐を除去 → その後で失効(npm トークンを最初に)。「ローテーションではなく失効」
  - SANS の結び: 「新しいのは認証情報の窃取ではなく、インストール不要の実行経路と、**こちらの対処の反射を引き金に変える**仕掛け」
  - 注意: SANS の著者は自身が関わるツールにも言及している(利益相反の開示あり)。仕掛けの記述自体は Socket・Wiz・Kodem の公開調査に基づく

出典:
- Wiz: https://www.wiz.io/blog/keyv-and-cacheable-npm-supply-chain-attack
- StepSecurity: https://www.stepsecurity.io/blog/chaindrop-npm-worm
- SANS ISC: https://isc.sans.edu/diary/33218

## PhantomRaven(依存が見えない)

- ✅ Koi Security が発見・命名。2025年8月に開始、**126 パッケージ・86,000 超のインストール**。公表時点で 80 超がまだ公開中(The Hacker News、The Register)
- 🟢 Sonatype が追加で 83 パッケージを特定し、計 200 超に。Adobe や Airbnb を装うもの、MCP サーバーを装うものもある(Sonatype)
- ✅ 手口は **Remote Dynamic Dependencies(RDD)**: 依存先を `packages.storeartifact[.]com` などの HTTP URL で指定し、npm がインストールのたびに外部サーバーから取得する。パッケージ本体には悪意あるコードがない
- ✅ Koi の研究者の言葉: 「npmjs.com はその URL を辿らない。セキュリティスキャナーも取得しない。依存解析ツールも無視する。**どの自動セキュリティシステムから見ても『依存 0』**」
- ✅ 取得された依存の**preinstall フック**でペイロードを実行 → install スクリプトのブロックで止まる。すり抜けたのは**検知**のほう
- ✅ パッケージ名は **slopsquatting**(LLM がハルシネーションで出す、もっともらしいが実在しない名前)を狙って登録
- 🟢 攻撃者は手口を段階的に進化させた: 同じパッケージの 1.0.0 は RDD なし、1.0.3 / 99.0.0 で RDD を使うようになった(Sonatype)
- 🟢 サーバー側で「ある利用者には無害なコード、別の利用者にはマルウェア」と出し分けられるため解析が難しい(Sonatype)
- ❌ Koi の原文は、現在 Palo Alto Networks のページへ転送され、本文が読めなかった
- ❌ 壁打ちメモの「10万件超」は確認できず。数字は「126 パッケージ・8.6万インストール」(初報)、「200 超」(Sonatype の追加後)

出典:
- Sonatype: https://www.sonatype.com/blog/phantomraven-npm-malware
- The Hacker News: https://thehackernews.com/2025/10/phantomraven-malware-found-in-126-npm.html
- The Register: https://www.theregister.com/2025/10/30/phantomraven_npm_malware/
- Koi(本文読めず): https://www.koi.ai/blog/phantomraven-npm-malware-hidden-in-invisible-dependencies

## chalk / debug(2025年9月、ブラウザ上の実行時)

- 🟢 2025-09-08 13:16 UTC に Aikido のフィードが検知。chalk・debug など**18 の人気パッケージ**、合計で**週20億ダウンロード超**(Aikido)
- 🟢 改ざんされたのは `index.js`。ペイロードは**ウェブサイトのクライアント側で実行される**、ブラウザ上のインターセプター。`fetch`・`XMLHttpRequest`・`window.ethereum` などのウォレット API をフックし、送金先や承認先を攻撃者のものに書き換える(Aikido)
  - → install スクリプトのブロックは関係ない。**サービス(エンドユーザー)側**を狙った例
- 🟢 原因はメンテナへの**フィッシング**。送信元ドメイン `npmjs.help` は攻撃の3日前(2025-09-05)に登録されていた(Aikido)
- ⚠️ 「5分で検知」「約2.5時間公開されていた」は二次情報のみ

出典:
- Aikido: https://www.aikido.dev/blog/npm-debug-and-chalk-packages-compromised

## 検知ツールの精度(論文)

論文: "Understanding NPM Malicious Package Detection: A Benchmark-Driven Empirical Analysis"(arXiv:2603.27549、2026年3月、プレプリント=査読前)

- 🟢 11 ツール・16 バリアントを、悪意ある 6,420 / 無害な 7,288 パッケージの共通ベンチマークで評価
- 🟢 **ML ベースのツールは時間とともに大きく劣化**: SAP_DT は 2021年の 87.15% → 2023年の 39.49%。ProfMal は 90.42% → 65.93%。Cerebro は 76.61% → 27.04%(2022 → 2024〜2025)。一方 Packj_static は全期間 97% 超、IntelGuard は 2021年以降 91% 超を維持
  - どのツールも検知パラメータは固定 → 差は**マルウェアの側の変化**によるもの
- 🟢 原因は「攻撃が高度化した(concept drift)」ではなく **concept convergence**: 難読化は減り、マルウェアは**単純になって**、特徴量の上で無害なコードと統計的に区別できなくなった
- 🟢 精度と再現率は、ツールが「コードの能力」と「悪意」の曖昧さをどう解くかで構造的に決まる(同じ API を正規のコードも悪意あるコードも使う)
- 🟢 **悪意あるパッケージの 80.3% は回避技術を使っていない**。エコシステムに公開前の必須スキャンがないため
- 🟢 ツールを組み合わせると精度 97.21%・F1 97.02% まで上がり、単体のどのツールより高い
- ⚠️ 査読前のプレプリント。数値を画面に出すなら「プレプリント」と付記する

出典:
- arXiv: https://arxiv.org/abs/2603.27549

## パッケージマネージャー間の仕様差(install スクリプト / リリース待機)

| | install スクリプトの扱い(既定) | リリース待機 |
|---|---|---|
| npm v12 | 🟢 `allowScripts` 既定オフ。依存の preinstall / install / postinstall と**暗黙の node-gyp ビルド**も止まる。`npm approve-scripts` で許可 | 🟢 `min-release-age`(日数、既定 null=無効) |
| pnpm | 🟢 `strictDepBuilds` 既定 true(未承認のビルドスクリプトがあると失敗)。`allowBuilds` で許可・拒否を指定。`ignoreScripts` は `.pnpmfile.mjs` の実行は止めない | 🟢 `minimumReleaseAge` 既定 **1440 分(1日)** |
| Yarn(Berry) | 🟢 `enableScripts` 既定 false(サードパーティの postinstall を実行しない。ワークスペース自身のものは実行) | 🟢 `npmMinimalAgeGate` |
| Bun | 🟢 「default-secure」: 許可リスト(`trustedDependencies`)にあるパッケージのライフサイクルスクリプトだけ実行 | 未確認 |

- ⚠️ 名前も粒度もバラバラ(allowScripts / allowBuilds / enableScripts / trustedDependencies)。総称としては「ライフサイクルスクリプトのブロック」
- ⚠️ 各ツールで暗黙の node-gyp ビルドが止まるかは、npm v12(公式明記)と pnpm 10.28.0(手元検証)以外は未確認

出典:
- npm docs: https://docs.npmjs.com/cli/v12/using-npm/config
- pnpm docs: https://github.com/pnpm/pnpm.io/blob/main/docs/settings/build.md / https://github.com/pnpm/pnpm.io/blob/main/docs/settings/dependency-resolution.md
- Yarn docs: https://yarnpkg.com/configuration/yarnrc
- Bun docs: https://bun.sh/docs/install/lifecycle

## pnpm の Minimum Release Age(仕様と抜け穴)

- 🟢 `minimumReleaseAge` 既定 **1440 分(1日)**。推移的依存を含む**すべての依存に適用**される → 「依存の依存には効かない」は誤解(pnpm docs)
- 🟢 **既定値のままだと「緩いモード」**: 条件を満たすバージョンがないとき、**条件を満たさない新しいバージョンにフォールバックしてインストールを続ける**(`minimumReleaseAgeStrict` の既定は、`minimumReleaseAge` を明示設定したときだけ true。組み込みの既定 1440 分は後方互換のため非 strict)(pnpm docs)
  - 11.1.3 の CHANGELOG: 緩いモードでは、未成熟なバージョンを `minimumReleaseAgeExclude` に**自動で追加**してインストールを進める(info メッセージが出る)
  - → **「デフォルトで1日待ってくれるから安心」は不正確**。明示的に設定してはじめて強制される
- 🟢 **11.1.3 で lockfile の再検証が入った**: 他所で弱いポリシーのもとで解決された lockfile(コミット済み・CI キャッシュ・古い pnpm 由来)が、公開直後のバージョンを**黙ってインストールすることはなくなった**(pnpm CHANGELOG)
  - → 壁打ちメモの「lockfile にポリシー違反があると素通りする問題は 11.1.3 で修正」は**正しかった**(前回「一致を確認できず」としたのは誤り。訂正)
- 🟢 **issue #10438 の原文(WebFetch で取得)**:
  - タイトル「`minimumReleaseAge` not enforced when dependency already exists in lockfile」。2026-01-10 作成(pnpm 10.27.0 での報告)
  - 内容: lockfile にすでに条件違反のバージョンが入っていると、`pnpm install`・`--fix-lockfile`・`--frozen-lockfile`・`--lockfile-only` のどれも**検査せずに通る**。新しく `pnpm add` する場合は正しく拒否される
  - **メンテナの最初の回答は「This seems to be by design.」**(2026-01-23)。lockfile が最新ならリゾルブを飛ばすので、チェックも飛ぶ、という説明。回避策として `--resolution-only` か lockfile の削除が案内された
  - その後、利用者から「少なくとも `--frozen-lockfile` では検査すべき」などの要望が続き、**2026-05-16 に PR #11583「fix: enforce minimumReleaseAge on existing lockfile entries」で修正され close**。メンテナが 05-18 に「🚢 11.1.3」とコメント
  - 11.1.3 に上げた利用者から「36 lockfile entries failed verification」のエラーが出たという報告もあり、既存 lockfile が実際に素通りしていたことがわかる
  - 「monorepo」「shared-workspace-lockfile」への言及はこの issue にはない
  - ⚠️ 解釈: **「設定したから守られている」と思っていた状態が、実は約4か月「仕様」として素通りしていた**例。設定の効き方を思い込まないこと、の具体例になり得る
- 🟢 `trustLockfile`(既定 false)を true にすると、lockfile の各エントリへの検証をスキップする。docs 自身が「外部の協力者が lockfile を編集できるなら false のままに」と警告
- 🟢 `sharedWorkspaceLockfile: false` については、`minimumReleaseAgeExcludePrune` などの後片付けがその構成で正しく動かない問題が修正されてきた(CHANGELOG)。⚠️ 壁打ちメモの「モノレポ構成での設定漏れ」に当たる具体的な抜け穴の報告は見つからなかった
- ⚠️ 解釈: Minimum Release Age 周りは **リリースごとに挙動の修正が続いている**。設定の意味が版によって変わり得る

出典:
- pnpm docs: https://github.com/pnpm/pnpm.io/blob/main/docs/settings/dependency-resolution.md
- pnpm CHANGELOG(11.x): https://unpkg.com/pnpm@11.28.1/CHANGELOG.md
- pnpm issue #10438: https://github.com/pnpm/pnpm/issues/10438

## 「攻撃者は開発者の環境を狙っている」

ユーザーの推測(開発者環境は資源を奪いやすく、人間へのソーシャルハッキングもしやすいので価値が高い)について:

- 🟢 **開発者環境・CI が主な標的になっている**ことは、今回読んだ一次情報でも一貫している
  - keyv: 「クローンしたフォルダを開くだけ」で実行。SANS は対象者に「リポジトリをクローンしたセキュリティエンジニア」も含まれると指摘
  - Phantom Gyp / keyv / AsyncAPI: 収集対象はクラウド鍵、CI のシークレット、npm / GitHub トークンなど**開発・配布のための認証情報**
  - ワーム型(Miasma、ChainDrop)は、盗んだ npm トークンで**次のパッケージを汚染する**。開発者の認証情報そのものが次の攻撃の足場になる
- ✅ 二次情報: Shai-Hulud 系の窃取マルウェアは開発者環境・CI/CD の 469 か所を走査(GitGuardian の報告を The Hacker News が引用)。AI ツールの認証情報や MCP サーバー設定も収集対象(SC World)
- 🟢 反例として押さえておく: chalk / debug は**エンドユーザーのブラウザ**(サービス側)を狙った。標的は開発者環境だけではない
- ⚠️ 「計算資源を奪いやすい」「ソーシャルハッキングしやすい」という**理由づけは裏が取れていない**。言えるのは「開発者の認証情報は、配布経路そのものへのアクセスになる(ワームの足場になる)」まで。理由を語るなら「推測」と明示する

出典:
- 上記各事例の出典
- The Hacker News: https://thehackernews.com/2026/09/shai-huluds-reach-just-grew-to-469.html
- SC World: https://www.scworld.com/news/new-mini-shai-hulud-attack-targets-npm-ecosystem

## 読めなかった一次情報

- Socket(node-ipc): curl では 403、WebFetch では名前解決エラー。どちらでも読めず
- Koi(PhantomRaven): 転送先で本文なし
- (Microsoft の AsyncAPI 記事と pnpm の issue #10438 は、curl では 403 だったが WebFetch で読めた)

## Node.js の標準機能で代用できる例(Node.js v26.13.1 公式ドキュメント、2026-11-17 確認)

| よく使われるパッケージ | Node.js の標準機能 | 追加されたバージョン |
|---|---|---|
| chalk(文字に色をつける) | 🟢 `util.styleText()` | v21.7.0 / v20.12.0 で追加。v23.5.0 / v22.13.0 で stable。v26.1.0 で16進カラー対応 |
| throttle / debounce 系 | 🟢 `util.throttle()` / `util.debounce()` | v26.10.0 で追加(⚠️ かなり新しい) |
| rimraf(ディレクトリの再帰削除) | 🟢 `fs.rm(path, { recursive: true })` | v14.14.0 で追加 |
| uuid | 🟢 `crypto.randomUUID()`(v7 は `crypto.randomUUIDv7()` もある) | v15.6.0 / v14.17.0 で追加 |

- 注意: `util.throttle` / `util.debounce` は v26.10.0 からなので、LTS で使えるかは聴衆の環境次第。例に出すなら「最近入った」と添える

出典:
- https://nodejs.org/api/util.html
- https://nodejs.org/api/crypto.html
- https://nodejs.org/api/fs.html

## Vite+(全部入りツールチェーン)

- ✅ VoidZero による MIT ライセンスのツールチェーン。Vite・Vitest・Oxlint・Oxfmt・Rolldown・tsdown とタスクランナーを、`vp` コマンド1つにまとめる
- ✅ 2026年にアルファ → ベータ(8月ごろ)→ 1.0(9〜10月ごろ。日付はソースで食い違い)
- ⚠️ 「VoidZero が Cloudflare に買収された」との報道あり(InfoQ)。一次未確認
- ⚠️ 二次情報のみ。スライドで具体的に触れるなら voidzero.dev の原文を確認する

出典:
- VoidZero(ベータ告知): https://voidzero.dev/posts/announcing-vite-plus-beta
- InfoQ(1.0): https://infoq.com/news/2026/10/cloudflare-viteplus-stable

## 一般によく挙げられている対策(使う側)(2026-11-17、ブログ・ガイドの横断)

複数の 2026 年のブログ・ガイドで共通して挙がっているもの(⚠️ いずれも二次情報。設定名・既定値は research.md 内の公式ドキュメント確認分を正とする):

| 対策 | 中身 | 挙げている例 |
|---|---|---|
| install スクリプトの停止 | `ignore-scripts=true`、許可リスト(npm `allowScripts` / pnpm `allowBuilds` など)。esbuild・sharp・Playwright など必要なものだけ許可 | OpenReplay、Arrange Act Assert、codercops ほか多数 |
| 公開直後のバージョンを避ける(クールダウン) | npm `min-release-age`(v11.10.0〜)、pnpm `minimumReleaseAge`、Yarn `npmMinimalAgeGate`。「数日待て」 | Classmethod、@IT(Mini Shai-Hulud の記事)、bodadotsh/npm-security-best-practices |
| lockfile の固定 | lockfile をコミットし、CI では `npm ci`。lockfile は意図しない更新を防ぎ、クールダウンは lockfile の汚染を防ぐ、と組み合わせで語られる | codercops、bastion ほか |
| Git / URL 依存の禁止 | `allow-git=none` など | OpenReplay ほか |
| 依存の検査・スキャン | `npm audit`、Socket などの PR スキャン | codercops、bastion |
| 公開経路の確認 | provenance バッジ、trusted publishing の有無 | 複数(使う側にとっては「選ぶ時の目安」) |

公開する側向け(この発表の聴衆には主ではない): trusted publishing(OIDC)、パスキー / ハードウェアキーの 2FA、CI(`pull_request_target` など)の堅牢化。

- 限界として繰り返し書かれていること: クールダウンは「数時間〜数日で消される侵害」しか捕まえられない。ignore-scripts は実行時のコードに埋め込まれた悪意は防げない(→ スライド 7・8 の論点と一致)

出典(二次):
- OpenReplay: https://blog.openreplay.com/ja/npm-サプライチェーン防御/
- Classmethod(min-release-age): https://dev.classmethod.jp/articles/npm-v11-min-release-age/
- Arrange Act Assert: https://arrangeactassert.com/posts/the-package-manager-settings-that-would-have-blocked-the-axios-attack/
- codercops: https://blog.codercops.com/blog/npm-supply-chain-hardening-checklist-2026/
- bastion: https://bastion.tech/blog/npm-supply-chain-attacks-2026-saas-security-guide
- npm-security-best-practices: https://github.com/bodadotsh/npm-security-best-practices
- @IT: https://atmarkit.itmedia.co.jp/ait/articles/2605/18/news019.html

## 推移的依存の例: eslint → … → keyv(2026-11-18、npm レジストリで確認)

- 🟢 各パッケージの最新版の `dependencies`(registry.npmjs.org/<name>/latest):
  - eslint 10.15.0 → file-entry-cache(`11.1.5 || >11.1.6 <12`)
  - file-entry-cache 11.1.5 → flat-cache(`^6.1.23`)
  - flat-cache 6.1.23 → cacheable(`^2.5.0`)
  - cacheable 2.5.0 → keyv(`^5.6.0`)
- → **eslint を入れただけで、2026年8月に侵害された keyv / cacheable の名前空間のパッケージが入る**
- 注意: これは「今の最新版」の依存関係。事件当時のバージョンでの経路は確認していない(当時は `eslint → file-entry-cache → flat-cache → keyv` だったとする資料もある)。スライドでは「今の依存関係だと」と言う

## pnpm の trustPolicy / blockExoticSubdeps(2026-11-20、pnpm docs 原文)

- 🟢 `trustPolicy`(既定 off / `no-downgrade`): 以前のリリースより信頼の証拠が弱くなった(例: trusted publisher → provenance のみ / なし)バージョンのインストールを失敗させる
- 🟢 `blockExoticSubdeps`(既定 true): Git リポジトリや tarball の URL といった「exotic」な取得元を使えるのは直接依存だけ。推移的依存はレジストリなど信頼できる取得元から解決させる

出典: https://github.com/pnpm/pnpm.io/blob/main/docs/settings/dependency-resolution.md
