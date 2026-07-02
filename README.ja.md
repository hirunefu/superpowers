# Superpowers

[English](README.md) | **日本語**

> これは [obra/superpowers](https://github.com/obra/superpowers) の個人フォークで、
> Claude Code 専用にカスタマイズしたものです。オリジナルの功績はすべて
> [Jesse Vincent](https://blog.fsck.com) 氏および
> [Prime Radiant](https://primeradiant.com) のメンバーに帰属します。

Superpowers は、コーディングエージェントのための完全なソフトウェア開発方法論です。組み合わせ可能なスキル群と、それをエージェントに確実に使わせるための初期インストラクションの上に成り立っています。

## クイックスタート

エージェントに Superpowers を与えましょう。Claude Code 向けの手順は [インストール](#インストール) を参照してください。

## 仕組み

すべてはコーディングエージェントを起動した瞬間から始まります。何かを作ろうとしていると気づくと、エージェントはいきなりコードを書き始めたりはしません。代わりに一歩引いて、あなたが本当は何をしたいのかを問い直します。

会話の中から仕様を引き出すと、実際に読んで理解できる程度の短いまとまりに分けて提示します。

設計に承認が出たあと、エージェントは実装プランを組み立てます。それは「センスが悪く、判断力がなく、プロジェクトの文脈を知らず、テストを嫌う、やる気だけはある新人エンジニア」でも辿れるほど明確なものです。真の red/green TDD、YAGNI（You Aren't Gonna Need It）、DRY を重視します。

次に、あなたが「go」と言えば、*subagent-driven-development*（サブエージェント駆動開発）プロセスが起動します。各エンジニアリングタスクをエージェントが処理し、その成果を検査・レビューしながら前進していきます。Claude が、あなたと組み立てたプランから逸れることなく、数時間にわたって自律的に作業できることも珍しくありません。

ほかにも色々ありますが、これがシステムの核心です。そしてスキルは自動的に発火するので、あなたは特別なことをする必要はありません。コーディングエージェントはただ Superpowers を備えているだけです。

## インストール

このフォークは独自の Claude Code マーケットプレイス（`superpowers-dev`）を同梱しています。
そのため、インストールすると上流のプラグインではなく **このカスタマイズ済みフォーク** が取り込まれます。

- このフォークのマーケットプレイスを登録します:

  ```bash
  /plugin marketplace add hirunefu/superpowers
  ```

- そこからプラグインをインストールします:

  ```bash
  /plugin install superpowers@superpowers-dev
  ```

> 未改変の上流プラグインや、他のハーネス（Antigravity / Codex / Cursor /
> Factory Droid / GitHub Copilot / Kimi Code / OpenCode / Pi）が必要ですか？
> [上流の README](https://github.com/obra/superpowers#installation) を参照してください。

## 上流との同期

`obra/superpowers` の最新の変更をこのフォークに取り込みます:

```bash
git fetch upstream
git merge upstream/main   # または: git rebase upstream/main
```

`upstream` リモートがまだ設定されていない場合:

```bash
git remote add upstream https://github.com/obra/superpowers.git
```

## 基本ワークフロー

1. **brainstorming** - コードを書く前に発火。質問を通じてラフなアイデアを洗練し、代替案を探り、検証のために設計をセクションごとに提示する。設計ドキュメントを保存する。

2. **using-git-worktrees** - 設計承認後に発火。新しいブランチ上に隔離されたワークスペースを作り、プロジェクトのセットアップを実行し、クリーンなテストのベースラインを検証する。

3. **writing-plans** - 承認済みの設計とともに発火。作業を一口サイズのタスク（各2〜5分）に分解する。すべてのタスクには正確なファイルパス、完全なコード、検証手順が含まれる。

4. **subagent-driven-development** または **executing-plans** - プランとともに発火。タスクごとに新鮮なサブエージェントを割り当て、二段階レビュー（仕様適合、次にコード品質）を行う。あるいは人間のチェックポイントを挟みながらバッチ実行する。

5. **test-driven-development** - 実装中に発火。RED-GREEN-REFACTOR を強制する: 失敗するテストを書き、失敗を確認し、最小限のコードを書き、成功を確認し、コミットする。テスト前に書かれたコードは削除する。

6. **requesting-code-review** - タスクの合間に発火。プランに照らしてレビューし、深刻度ごとに問題を報告する。クリティカルな問題は進行をブロックする。

7. **finishing-a-development-branch** - タスク完了時に発火。テストを検証し、選択肢（マージ / PR / 保持 / 破棄）を提示し、ワークツリーを片付ける。

**エージェントはどんなタスクの前にも、関連するスキルがないか確認します。** これらは提案ではなく必須のワークフローです。

## 中身

### スキルライブラリ

**テスト**
- **test-driven-development** - RED-GREEN-REFACTOR サイクル（テストのアンチパターン集を含む）

**デバッグ**
- **systematic-debugging** - 4 フェーズの根本原因究明プロセス（root-cause-tracing、defense-in-depth、condition-based-waiting の各テクニックを含む）
- **verification-before-completion** - 本当に直っているかを確認する

**コラボレーション**
- **brainstorming** - ソクラテス式の設計洗練
- **writing-plans** - 詳細な実装プラン
- **executing-plans** - チェックポイント付きのバッチ実行
- **dispatching-parallel-agents** - 並行サブエージェントのワークフロー
- **requesting-code-review** - レビュー前チェックリスト
- **receiving-code-review** - フィードバックへの対応
- **using-git-worktrees** - 並行開発ブランチ
- **finishing-a-development-branch** - マージ / PR 判断ワークフロー
- **subagent-driven-development** - 二段階レビュー（仕様適合、次にコード品質）による高速イテレーション

**メタ**
- **writing-skills** - ベストプラクティスに従って新しいスキルを作る（テスト方法論を含む）
- **using-superpowers** - スキルシステムの入門

## 哲学

- **テスト駆動開発** - 常にテストを先に書く
- **場当たり的より体系的に** - 推測ではなくプロセスを
- **複雑さの低減** - シンプルさを第一の目標に
- **主張より証拠** - 成功を宣言する前に検証する

[オリジナルのリリースアナウンス](https://blog.fsck.com/2025/10/09/superpowers/) を読む。

## コントリビュート

これは個人フォークです。プロジェクトへのコントリビュートは上流で行ってください。
[obra/superpowers のコントリビューションガイド](https://github.com/obra/superpowers#contributing) を参照してください。

## ライセンス

MIT License - 詳細は LICENSE ファイルを参照してください。

## ビジュアルコンパニオンのテレメトリ

スキルやプラグインは作者に何のフィードバックも返さないため、私たちにはどれだけの人が Superpowers を使っているのか知るすべがありません。デフォルトでは、brainstorming のオプション機能であるビジュアルコンパニオンに表示される Prime Radiant のロゴは、私たちのウェブサイトから読み込まれます。そこには使用中の Superpowers のバージョンが含まれます。あなたのプロジェクト、プロンプト、コーディングエージェントに関する情報は一切含まれません。クリックの内容や、あなたが何を作っているかを私たちが見ることはありません。これにより、どれくらいの人がどのバージョンの Superpowers を使っているのかを大まかに把握できます。これは 100% オプションです。無効にするには、環境変数 `SUPERPOWERS_DISABLE_TELEMETRY` に任意の真値を設定してください。Superpowers は Claude Code の `DISABLE_TELEMETRY` および `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` によるオプトアウトにも従います。

## コミュニティ

Superpowers は [Jesse Vincent](https://blog.fsck.com) 氏と [Prime Radiant](https://primeradiant.com) のメンバーによって作られています。

- **Discord**: コミュニティサポート、質問、Superpowers で作っているものの共有は [こちらから参加](https://discord.gg/35wsABTejz)
- **Issues**: https://github.com/obra/superpowers/issues
- **リリース告知**: 新バージョンの通知を受け取るには [サインアップ](https://primeradiant.com/superpowers/)
