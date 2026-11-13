# research

braindump.md の主張・事例の裏取り結果。出典つき。
調査日: 2026-11-13。いずれも WebSearch の要約ベースで、**一次情報(原文)の確認は未了**。スライドに数値を載せる前に原文を当たること。

凡例: ✅ 複数ソースで一致 / ⚠️ ソース間で食い違い・一部のみ確認 / ❌ 確認できず・壁打ちメモと食い違い / ⬜ 未調査

---

## node-ipc(import 時に発火)

- ✅ 2026-05-14 に悪意ある3バージョン(9.1.6 / 9.2.3 / 12.0.1)が公開された
- ✅ ペイロードは CommonJS エントリポイントの末尾に難読化して追加され、`require('node-ipc')` のたびに無条件で動く。npm のライフサイクルスクリプトには依存しない
- ✅ 90 超の認証情報カテゴリを狙い、DNS トンネリング等で外部へ送信したとの報告
- ✅ 経路は、休眠メンテナのアカウント乗っ取りの可能性(Socket の見立て。Snyk は「証拠ではなく手がかり」とする)
- ✅ クリーンなバージョンは 9.2.1 / 12.0.0
- ⚠️ 「Socket が約3分で検知」: 今回の検索結果では確認できなかった。原文要確認
- ⚠️ ダウンロード数(週 約82万 / 1,000万超)は食い違ったまま。数値を使うなら npm の統計で確認

出典:
- Socket: https://socket.dev/blog/node-ipc-package-compromised
- The Hacker News: https://thehackernews.com/2026/05/stealer-backdoor-found-in-3-node-ipc.html
- Snyk: https://snyk.io/blog/malicious-node-ipc-versions-published-npm

## Phantom Gyp(binding.gyp 経由の暗黙ビルド)

- ✅ 2026-06-03 に 57 パッケージ・286 バージョン超。2時間未満で完了したとの報告
- ✅ ライフサイクルスクリプトを一切使わず、`binding.gyp` があると npm が暗黙で `node-gyp rebuild` を走らせる挙動を利用。解析された binding.gyp は 157 バイト
- ✅ 背後は **Miasma ワーム**。Bun ベースの認証情報窃取、CI/CD・クラウド認証情報を狙い、他のメンテナのパッケージを再公開して自己増殖
- ⚠️ 命名は StepSecurity とも JFrog とも書かれている
- 注意: 「Phantom Gyp」は手口の名前、キャンペーン / ワームの名前は「Miasma」

出典:
- Chainguard: https://www.chainguard.dev/de-DE/unchained/chainguard-artifacts-safe-from-miasma-phantom-gyp-npm-attack
- Cyber Security News: https://cybersecuritynews.com/binding-gyp-supply-chain-attack-compromises-dozens-of-npm-packages/
- SOC Prime: https://socprime.com/active-threats/miasma-supply-chain-attack-spreads-through-the-phantom-gyp-worm
- Cryptika: https://www.cryptika.com/binding-gyp-supply-chain-attack-compromises-dozens-of-npm-packages-across-maintainer-accounts/

## npm v12

- ✅ 依存パッケージのライフサイクルスクリプト(preinstall / install / postinstall)と、**暗黙の node-gyp rebuild** が、`allowScripts` で許可しない限り動かなくなる(デフォルトでオフ)
- ✅ `npm approve-scripts` / `npm deny-scripts` で管理。ブロック時は警告が出るがインストールは完了する
- ⚠️ リリース日 2026-07-08 は DEV Community の1記事のみ。The Hacker News は 2026/07 の記事、InfoQ は 2026/08 の URL。**正確な日付は公式で確認**
- 「Phantom Gyp の約1ヶ月後に塞がれた」という時系列は、6/3 → 7月、で整合はするが、v12 のこの変更が Phantom Gyp への対応として入ったかは未確認(v12 の計画自体はそれ以前から報じられている: gigazine 2026-06-11 の記事など)。**「対応して塞いだ」と言い切らない**

出典:
- The Hacker News: https://thehackernews.com/2026/07/npm-12-disables-install-scripts-by.html
- Semgrep: https://semgrep.dev/blog/2026/rip-npm-postinstall-scripts-npm-v12-default-change
- InfoQ: https://www.infoq.com/news/2026/08/npm-12-released/
- npm/cli issue: https://github.com/npm/cli/issues/9424

## AsyncAPI(Provenance 付きで侵害)

- ✅ 2026-07-14。@asyncapi の4パッケージ・5バージョン(generator 3.3.1、generator-helpers 1.1.1、generator-components 0.7.1、specs 6.11.2-alpha.1 / 6.11.2)
- ✅ 正規の GitHub Actions リリースワークフロー + npm の OIDC Trusted Publisher 経由で公開されたため、**Provenance は本物**
- ✅ ペイロードは `require()` 時に動く(インストール時ではない)。Miasma RAT / 情報窃取
- ✅ 攻撃者は next ブランチへの push 権限を得て、プロジェクト自身のリリースパイプラインを使った(37個のおとり PR の中に1本、という報告もある)
- ⚠️ npm トークンが盗まれたかは調査機関で食い違い(StepSecurity: 不要 / Aikido: 盗まれた)
- ⚠️ specs のバージョンは 6.11.1 と書くソースもある
- → 壁打ちメモの「Provenance 付きのまま侵害」は裏が取れた。「Trusted Publisher でも防げない」の例として使える

出典:
- Microsoft: https://microsoft.com/en-us/security/blog/2026/07/15/unpacking-asyncapi-npm-supply-chain-compromise-import-time-payload-delivery
- Semgrep: https://semgrep.dev/blog/2026/miasma-v3-asyncapi-npm-hardening
- BleepingComputer: https://bleepingcomputer.com/news/security/-asyncapi-npm-packages-infected-with-credential-stealing-malware
- StepSecurity: https://stepsecurity.io/blog/compromised-next-branch-pushes-malicious-asyncapi-generator-generator-helpers-and-generator-components-to-npm
- JFrog: https://research.jfrog.com/post/miasma-worm-returns-to-npm/

## keyv(AI エージェント / IDE 設定ファイル)

- ✅ 2026年8月。keyv リポジトリに、`.claude/settings.json` の SessionStart フックと `.vscode/tasks.json` の folderOpen タスクが悪意ある commit で仕込まれた。**リポジトリを開くだけで** `npm install` なしに動き得る
- ✅ 2つのフックは互いの設定ディレクトリのスクリプトを呼び合い、片方を消しても連鎖が止まらない(Upwind)
- ✅ Claude / OpenAI / Codex / Cursor / Gemini 等の AI エージェントの認証情報ストアも収集対象
- ✅ 侵害されたパッケージは有効な Provenance 付きで公開された(StepSecurity)
- ⚠️ 規模は食い違い(StepSecurity: 444 パッケージ / 2,212 バージョン、別ソース: アカウントから11リリース)
- ⚠️ **「ローテーションした瞬間に発火するトラップ」は SANS ISC の記述1件のみ**。仕組みは未確認。使うなら SANS / Wiz の原文を確認
- ❌ 壁打ちメモの「Wave Six」という呼称は、今回のソースには出てこなかった。ソース上の名前は「ChainDrop」「Shai-Hulud の再燃」「Mini Shai-Hulud」など。**呼称は使わず、「keyv の事件」で行く方が安全**

出典:
- Cycode: https://cycode.com/blog/keyv-cacheable-npm-worm-ai-coding-agents/
- StepSecurity: https://stepsecurity.io/blog/chaindrop-npm-worm
- Upwind: https://upwind.io/feed/keyv-supply-chain-compromise-an-npm-worm-that-takes-its-orders-from-an-ethereum-smart-contract
- SANS ISC: https://mail2.dshield.org/diary/33218
- Mend: https://www.mend.io/blog/keyv-malicious-packages

## pnpm の Minimum Release Age

- ⚠️ 11.1.3 のリリースノートで #10438 が閉じられているが、内容は「`minimumReleaseAge` を明示設定したら `minimumReleaseAgeStrict` を自動で有効にする」変更。壁打ちメモの「lockfile にポリシー違反のバージョンがあると素通りする問題を修正」とは**一致が確認できなかった**
- ✅ 11.3 で `trustLockfile`(lockfile の各エントリに対して `minimumReleaseAge` を再適用するかの設定)が入った。こちらが「lockfile 素通り」に近い話かもしれない
- ✅ 11.4 に、バイパス時に警告を出す仕組み(`--force` や `pnpm update` は整合性チェックをバイパスしない)
- ✅ 12.0.0-rc.4 に `sharedWorkspaceLockfile: false` に関する記述がある
- ❌ `shared-workspace-lockfile = false` でのバイパスの具体的な報告は、今回の検索では見つからなかった
- ⬜ 「デフォルト1日(pnpm v11)」は未確認
- → **Minimum Release Age の「抜け穴」を話す場合は、pnpm の issue / リリースノート原文を確認してから**。確認できるまでは「設定の組み合わせ次第で意図せず無効になり得る」という抽象度に留める

出典:
- pnpm 11.1.3: https://newreleases.io/project/npm/pnpm/release/11.1.3
- pnpm 11.3: https://pnpm.io/blog/releases/11.3
- pnpm 11.4: https://pnpm.io/blog/releases/11.4
- pnpm 12.0.0-rc.4: https://newreleases.io/project/github/pnpm/pnpm/release/v12.0.0-rc.4
- 解説: https://lilting.ch/en/articles/pnpm-11-minimum-release-age-default-shai-hulud-defense

## 「攻撃者は開発者の環境を狙っている」

ユーザーの推測(開発者環境は資源を奪いやすく、人間へのソーシャルハッキングもしやすいので価値が高い)に対して:

- ✅ **「開発者環境・CI が狙われている」は裏が取れる**
  - Shai-Hulud 系の窃取マルウェアは、開発者環境・CI/CD ツールの 469 か所を走査(GitGuardian の報告を The Hacker News が引用)
  - 開発者ワークステーションには Git にコミットされていない秘密情報があり、1つのトークンがソースコードへのアクセスにつながり得る
  - SAP の npm パッケージ事件では、VS Code や Claude Code の設定ファイルによる永続化も試みられた
  - AI ツールの認証情報と MCP サーバー設定の収集を狙った最初のペイロード、との報告(SC World)
  - Mini Shai-Hulud(TeamPCP の仕業とされる)は PyPI・npm・PHP を横断し、1,800 超の開発者リポジトリの窃取情報を確保。VPN 認証情報、暗号資産ウォレット、Slack/Discord のセッショントークンも対象
  - Shai-Hulud 2.0 は 600 超のパッケージに感染、2万超の GitHub リポジトリを窃取データの置き場に使い、1.5万件の秘密情報が漏れた。長期有効なトークンと CI 環境が弱点(Riptides の事後分析)
- ❌ 「計算資源の奪取がしやすい」「ソーシャルハッキングがしやすい」という**理由の部分は、今回のソースでは裏が取れていない**。「狙われている」という事実と、「なぜ狙うか」の解釈は分けて扱う
- 1回の検索で、各社ブログ・転載記事が中心。業界横断の権威ある調査レポートは見つかっていない

出典:
- The Hacker News: https://thehackernews.com/2026/09/shai-huluds-reach-just-grew-to-469.html
- CSO Online: https://www.csoonline.com/article/4165420/sap-npm-package-attack-highlights-risks-in-developer-tools-and-ci-cd-pipelines.html
- SC World: https://www.scworld.com/news/new-mini-shai-hulud-attack-targets-npm-ecosystem
- Security Boulevard: https://securityboulevard.com/2026/05/1800-developers-hit-in-mini-shai-hulud-supply-chain-attack-across-pypi-npm-and-php/
- Backslash: https://www.backslash.security/blog/shai-huluds-new-frontier

## 未調査

- ⬜ PhantomRaven(2025年秋、「10万件超」等の数値)
- ⬜ chalk/debug 事件(2025年9月)
- ⬜ 検知ツール(ML ベース)の精度劣化の数値(87% → 39% など)。元論文が必要
- ⬜ `allowScripts` / `onlyBuiltDependencies` / Bun の `trustedDependencies` の仕様の差
