---
title: "Claude Code Skills × launchd で「寝ている間に仕事が終わる」仕組みを作った"
emoji: "🌙"
type: "tech"
topics: ["claudecode", "skills", "launchd", "automation", "macos"]
published: false
---

## はじめに

朝6時半にアラームで目が覚める。まだ布団の中でスマホを開くと、メールが1通届いている。

差出人は自分のMac。件名は「AXIS デイリーレポート」。中を開くと、入札案件3件の調査結果、補助金6種類の公募状況、JTBの最新ニュース、Azure App Serviceのアップデート情報、そしてそれらを統合した戦略レポートが入っている。

全部、深夜3時にMacが自動起床して、Claude Codeが勝手にやってくれたものだ。

前回の記事では5層自動化の全体像を紹介した。今回はその中核となる「夜間ワーカー（Layer 1）」を深掘りする。Skills（Claude Codeの拡張機能）とlaunchd（macOSのスケジューラ）を組み合わせて、寝ている間にAIが情報収集・分析・レポート生成まで完了する仕組みだ。

## night_workerの全体構成 — 7つのPhase

夜間ワーカーは7つのPhaseを順に実行するパイプライン型のシェルスクリプトだ。前段の出力が次段の入力になる。

```
3:00  ┌──────────────────────────────────────┐
      │  Phase 1: データ収集（wave方式 3-5並列） │
      │  入札/補助金/JTB/Azure/セキュリティ      │
      ├──────────────────────────────────────┤
      │  Phase 2: 状況評価 + ブリーフィング統合    │
      │  Phase 1の全出力 → 1つのレポートに統合    │
      ├──────────────────────────────────────┤
      │  Phase 3: 戦略深堀り（3並列）             │
      │  JTB分析 / プロダクト戦略 / 事業開発      │
      ├──────────────────────────────────────┤
      │  Phase 4: 戦略統合 + マネージャー処理      │
      │  改善ループ / 俯瞰レビュー / アクション抽出│
      ├──────────────────────────────────────┤
      │  Phase 4.5: セカンドパス（余力で品質向上）  │
      ├──────────────────────────────────────┤
      │  Phase 5: モーニングレポート生成            │
      ├──────────────────────────────────────┤
      │  Phase 6: ファイルローテーション（30日超削除）│
      ├──────────────────────────────────────┤
      │  Phase 7: 自動エスカレーション              │
6:45  └──────────────────────────────────────┘
```

Phase 1でデータを集め、Phase 2で状況を評価し、Phase 3で3つの視点から深堀りし、Phase 4で統合する。Phase 5でメール用のレポートにまとめ、Phase 6で古いファイルを掃除し、Phase 7でエラーがあればmacOS通知でアラートを飛ばす。

全体で約45分。3:00に開始して6:45までに終わる。

## Skills（CLAUDE.md拡張）の活用法

### CLAUDE.mdとSkillsの使い分け

Claude Codeにはプロジェクトのルールや文脈を伝える方法が2つある。

- **CLAUDE.md**: プロジェクトルート直下に置く設定ファイル。プロジェクト全体に適用されるルール、制約、ディレクトリ構成を書く
- **Skills**: `.claude/skills/` 配下にMarkdownで定義する再利用可能な機能単位。特定のトリガーで発動し、決まった手順を実行する

自分の使い分けはこうだ。

- CLAUDE.md → 「このプロジェクトでは常にAzureを前提にしろ」「コミットメッセージは日本語」といったグローバルルール
- Skills → 「おはようと言われたらカレンダーから今日の予定を取得して15分刻みの工程表を作れ」といった具体的な手順

Skillsの中身はただのMarkdownだ。例えばスケジュール生成Skillはこう書く。

```markdown
# Skill: daily-schedule

「おはよう」「今日の予定」で自動発動。
Google Calendarから予定を取得し、タスクの優先度分類と
15分刻みの工程表を生成する。

## トリガー
- 「おはよう」「おは」
- 「今日の予定」「今日何ある？」

## 実行手順
1. Google Calendar MCPで今日の予定を取得
2. メモリから未実装タスクと進行中タスクを読み込む
3. 優先度分類（緊急+重要 / 重要 / ルーティン / 低優先度）
4. 15分刻みの工程表を生成して表示
```

これだけで、毎朝「おはよう」と打つだけでその日の最適なスケジュールが出てくるようになる。

### npx skills findで探す

コミュニティが公開しているSkillsは`npx skills find`で検索できる。

```bash
npx skills find "python"
# → python-development, pytest-runner, etc.

npx skills install vercel-labs/skills/find-skills
# → .claude/skills/find-skills/ にインストールされる
```

### 自作Skillsとの組み合わせ

現在6つのSkillsを運用している。外部から3つ、自作が3つだ。

外部Skills:
- `find-skills` — Skillsの検索・インストール
- `python-development` — Python開発支援
- `skill-creator` — 新規Skill作成の対話的ウィザード

自作Skills:
- `daily-schedule` — カレンダー連携の日次スケジュール生成
- `agent-memory` — 「覚えておいて」「思い出して」で発動する記憶管理
- `diagram` — 「図解して」で発動するHTML→PNG図解生成

自作Skillsのポイントは、自然言語トリガーと組み合わせることだ。`.claude/rules/triggers.md`に「おはよう→daily-schedule発動」「覚えておいて→agent-memory発動」と書いておくと、ユーザーが特別なコマンドを覚える必要がない。普通に話しかけるだけでSkillが発動する。

## launchdで毎日3:00に自動実行

### plistの書き方

macOSでcron的なスケジュール実行をするにはlaunchdを使う。`~/Library/LaunchAgents/`にplistファイルを置くだけ。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.myproject.night-worker</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/path/to/scripts/night_worker.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>3</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/path/to/logs/night-worker.log</string>
    <key>StandardErrorPath</key>
    <string>/path/to/logs/night-worker.log</string>
    <key>WorkingDirectory</key>
    <string>/path/to/project</string>
    <key>ExitTimeOut</key>
    <integer>14400</integer>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
        <key>HOME</key>
        <string>/Users/yourname</string>
    </dict>
</dict>
</plist>
```

`ExitTimeOut`を14400秒（4時間）にしているのがポイント。Claude Codeの並列処理は予想以上に時間がかかることがあるので、余裕を持たせる。

登録は1コマンド。

```bash
launchctl load ~/Library/LaunchAgents/com.myproject.night-worker.plist
```

### pmsetで自動起床

launchdはMacが起動していないと動かない。深夜3時にMacがスリープしていたら何も起きない。そこで`pmset`でスケジュール起床を設定する。

```bash
# 毎日2:55に自動起床（3:00の5分前に余裕を持って）
sudo pmset repeat wakeorpoweron MTWRFSU 02:55:00
```

`MTWRFSU`は月火水木金土日の全曜日。ワーカーの5分前に起こしておくことで、ネットワーク接続やディスクのスピンアップが完了した状態でスクリプトが走り出す。

### caffeinateでスリープ防止

起床してもユーザー操作がなければmacOSは再びスリープしようとする。これを防ぐのが`caffeinate`だ。

```bash
# 3時間45分（3:00→6:45）スリープを防止
caffeinate -u -s -t 13500 &
CAFFEINATE_PID=$!
```

`-u`はディスプレイの電源管理を上書き、`-s`はシステムスリープを防止。`-t 13500`で13500秒（225分）後に自動解除される。6:45に次のワーカー（チャンネル収集）が引き継ぐまで起きていれば良い。

## 実際の成果物

### モーニングレポートの中身

毎朝生成されるレポートの構造はこうなっている。

```markdown
# デイリーレポート (2026-03-15)

全体ステータス: 正常完了

## 夜間ワーカー実行ダッシュボード
- 実行時間: 03:00 〜 03:47（47分）

| Phase | 内容 | 所要時間 | 結果 |
|-------|------|----------|------|
| P1 | データ収集（4タスク） | 18分32秒 | 0件失敗 |
| P2 | 状況評価 | 6分15秒 | 完了 |
| P3 | 戦略深堀り | 9分44秒 | 0件失敗 |
| P4 | 戦略統合+マネージャー | 8分20秒 | 0件失敗 |

## ブリーフィング + 状況評価
（入札案件、競合動向、Azureアップデートの要約）

## 戦略レポート
（エグゼクティブサマリー、今日のアクション、リスク分析）
```

実際に生成されるファイル群はこんな感じだ。

```
daily_news/
├── 2026-03-15_bidding.md          # 入札案件調査
├── 2026-03-15_grants.md           # 補助金6種の公募状況
├── 2026-03-15_jtb_asiangames.md   # JTB・アジア大会動向
├── 2026-03-15_azure_updates.md    # Azure最新情報（月・木）
├── 2026-03-15_competitor.md       # 競合調査（月曜のみ）
├── 2026-03-15_morning_report.md   # 統合レポート
└── strategy/
    ├── 2026-03-15_step1_situation.md  # 状況評価
    ├── 2026-03-15_step2_jtb.md       # JTBパートナーシップ分析
    ├── 2026-03-15_step2_product.md   # プロダクト戦略
    ├── 2026-03-15_step2_business.md  # 事業開発分析
    └── 2026-03-15_step3_strategy.md  # 統合戦略レポート
```

### 戦略レポートが30本以上自動生成された話

この仕組みを1ヶ月動かした結果、`daily_news/strategy/`には30本以上の戦略レポートが蓄積された。1本あたり平均3,000〜5,000字。人間が毎日これを書いたら1日2時間はかかる作業だ。

ここで重要なのが「前日の戦略レポートを翌日に引き継ぐ」仕組みだ。Phase 2の状況評価時に前日のレポートを入力に含めている。

```bash
# 前日の戦略レポートを探す
PREV_STRATEGY=$(ls -t "$STRATEGY_DIR"/${YESTERDAY}_step3_strategy.md \
  2>/dev/null | head -1)

# Phase 2のプロンプトに追加
PREV_STRATEGY_REF=""
if [ -n "$PREV_STRATEGY" ] && [ -f "$PREV_STRATEGY" ]; then
  PREV_STRATEGY_REF="前日の戦略レポート: ${PREV_STRATEGY}"
fi
```

これにより「先週は入札案件が2件だったが今週は5件に増えた」「競合のA社が新機能をリリースした。先月の予測が当たった」といった時系列の分析ができるようになる。蓄積が価値になる。

## 改善ループ — 品質スコア64点→86点

夜間ワーカーが生成したコンテンツの品質を自動で上げる仕組みが`improvement_loop.sh`だ。

### 仕組み

3人の仮想レビュアーが並列でレビューし、フィードバックを統合して改善版を生成する。これを収束するまで最大5回繰り返す。

```bash
bash scripts/improvement_loop.sh --preset note -i article.md
```

プリセットごとにレビュー基準とレビュアーの視点が定義されている。

```
プリセット    レビュアー
note         読者目線の編集者 / SEO専門家 / 対象分野の専門家
strategy     COO / CFO / CTO
x-post       SNSマーケター / ターゲットユーザー / ブランドマネージャー
script       シニアDevOps / セキュリティエンジニア / SRE
```

### スコアの推移

Note記事の品質基準は8項目100点満点で採点される。即REJECT条件（体験ゼロ、説教臭い着地、固有名詞漏れ）に該当すると70点未満確定。

改善ループ導入前の初期生成品質は平均64点。REJECT級だった。タイトルが「〜の方法」パターンだったり、リード文が「今回は〜について紹介します」で始まったり、着地が「参考になれば幸いです」だったりした。

改善ループを2回回すと平均86点まで上がった。主な改善点は以下の通り。

- タイトルに逆説や数字が入るようになった
- リード文が体験ベースになった（「3時間かかっていた作業が15分になった」等）
- 段落の冒頭にミニフック（「ここからが本題だ」「多くの人がここで間違える」）が挿入されるようになった

22点の改善が完全自動で起きる。人間は何もしていない。

### 夜間ワーカーへの組み込み

Phase 4で戦略レポートの改善ループを非同期実行している。

```bash
# 時間に余裕がある場合のみ改善ループを起動
if [ -f "${STRATEGY_DIR}/${DATE}_step3_strategy.md" ] && \
   [ $ELAPSED -lt 2700 ]; then
  bash scripts/improvement_loop.sh \
    --preset strategy \
    -i "${STRATEGY_DIR}/${DATE}_step3_strategy.md" \
    -m 2 -t 600 --task-timeout 180 &
  STRATEGY_LOOP_PID=$!
fi
```

`-m 2`で最大2回の反復、`-t 600`で全体10分の制限をかけている。夜間ワーカーの残り時間を食いつぶさないようにするためだ。

## ハマりポイント

### claude -pの入れ子実行

Claude Codeの中から`claude -p`を呼ぶ（スクリプト経由の再帰実行）には、環境変数`CLAUDECODE`をunsetする必要がある。これを忘れるとプロセスが即座にエラーで落ちる。

```bash
# スクリプトの先頭に必ず書く
unset CLAUDECODE 2>/dev/null || true
```

### gtimeout必須

macOSの標準`timeout`コマンドは存在しない。GNU coreutilsの`gtimeout`をインストールする必要がある。

```bash
brew install coreutils
```

各`claude -p`にタイムアウトを設定しないと、1つのタスクが無限に走り続けて全体が詰まる。10分のタイムアウトに加えて、`--kill-after=30`でSIGKILLのフォールバックも入れている。

```bash
gtimeout --kill-after=30 600 claude -p \
  --permission-mode bypassPermissions \
  "プロンプト" \
  --allowedTools "WebSearch,Read,Write"
```

### 並列実行のレート制限

Claude Pro Maxでも、`claude -p`を一気に10並列で走らせるとレート制限に引っかかることがある。最初はPhase 1で11タスクを一斉に起動していたが、5つ目あたりから応答が返ってこなくなる現象が頻発した。

対策として「wave方式」を導入した。タスクを3-4件ずつのwaveに分けて順次実行する。

```bash
# Wave 1: 高優先度（3-5タスク）
run_wave 1 "${P1_WAVE1[@]}"

# Wave 2: コンテンツ（2-3タスク）
run_wave 2 "${P1_WAVE2[@]}"

# Wave 3: リサーチ（1-2タスク）
run_wave 3 "${P1_WAVE3[@]}"
```

Wave間は前のwaveの完了を待ってから次を実行する。1 waveあたり3-5並列なら安定して動く。

### Phase全体のデッドライン制御

タスク個別のタイムアウトだけでは不十分だった。Phase 1全体で60分を超えたら、残りのタスクを強制終了して次のPhaseに進む仕組みが必要になった。

```bash
PHASE1_DEADLINE=$((START_TIME + 3600))  # 60分

wait_named_with_deadline() {
  # 各タスクの完了を待ちつつ、デッドラインを超えたら強制kill
  while kill -0 "$pid" 2>/dev/null; do
    if [ "$(date +%s)" -ge "$deadline" ]; then
      kill "$pid" 2>/dev/null || true
      break
    fi
    sleep 1
  done
}
```

### Clamshell Sleepとの戦い

MacBookの蓋を閉じた状態（クラムシェルモード）だと、pmsetで起こしてもすぐスリープに戻ることがある。SIGTERMが飛んでスクリプトが途中で死ぬ。

対策として、SIGTERMハンドラでログに記録するようにした。

```bash
handle_sigterm() {
  echo "[SIGTERM受信] Clamshell Sleepの可能性" >> "$LOG_FILE"
  cleanup
  exit 143
}
trap handle_sigterm TERM
```

根本解決は「蓋を開けたまま寝る」か、外部ディスプレイに接続しておくこと。自分はデスクの上に置いて蓋を開けっぱなしにしている。

### lockファイルによる二重起動防止

launchdは前回の実行が終わっていなくても、次の実行時刻が来たら新しいプロセスを起動することがある。二重起動を防ぐためにlockファイルを使う。

```bash
LOCK_FILE="$LOG_DIR/night_worker.lock"

if [ -f "$LOCK_FILE" ]; then
  LOCK_PID=$(cat "$LOCK_FILE" 2>/dev/null)
  if kill -0 "$LOCK_PID" 2>/dev/null; then
    echo "既に実行中（PID: $LOCK_PID）。終了。"
    exit 0
  fi
fi
echo $$ > "$LOCK_FILE"
```

単にファイルの存在だけでなく、PIDが生きているかまで確認する。前回の実行が異常終了してlockファイルが残ったままでも、次の実行がブロックされない。

## まとめ — 「放置力」が生産性になる

1人で事業を回していると、「やるべきこと」は無限に湧いてくるのに「やれる時間」は1日24時間しかない。この構造的な問題に対する自分の答えが「放置力」だ。

PCを放置している時間——寝ている間、移動中、会議中——を全部AIの作業時間に変える。人間がやるべきことは、朝起きてレポートを読み、判断が必要なものだけ判断すること。

night_workerの運用で分かったことを3つ挙げる。

- 「完璧な自動化」は目指さなくていい。7-8割の精度でも、毎日自動で走ることに価値がある。足りない部分は朝10分で補えばいい
- パイプライン型設計にすると、各Phaseを独立して改善できる。Phase 1のタスクを増やしても、Phase 2以降は変更不要
- 蓄積が価値になる。1日分のレポートは大したことないが、30日分が溜まると競合の動向変化やトレンドが見えてくる

Claude Code + Skills + launchd。この組み合わせで「寝ている間に仕事が終わる」は現実になった。

---

次回の記事では、チャンネルエンジンの設計とYouTube RSSフィードの活用について掘り下げる予定。

自動化環境のテンプレートはGitHubで公開中。

https://github.com/Daidai-inc/claude-automation-package
