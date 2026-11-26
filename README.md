# Template
新規プロジェクト用のテンプレートリポジトリ。Node.js + Jest によるテスト、GitHub Actions による CI/CD、Issue/PR テンプレート、Dependabot をあらかじめ設定済み。

## セットアップ
```bash
npm install
```

## テスト
```bash
npm test
```

## 構成
### サンプルコード（実装に合わせて削除・置き換える）
- `sum.js` — Jest の動作確認用サンプル実装
- `sum.test.js` — 上記に対応するサンプルテスト

### 雛形（そのまま利用する土台）
- `package.json` — npm scripts・依存関係の定義
- `.github/workflows/ci.yml` — main への push・PR 時にテストを実行
- `.github/workflows/cd.yml` — main へのマージ後、**承認を待ってから**本番デプロイ（`deploy.sh` を用意して利用）。承認した版に `release-<番号>` のタグを付け、GitHub Projects の列も動かす
- `.github/scripts/move-released-to-done.sh` — 承認後に、GitHub Projects の「承認待ち」の PR を「本番反映済み」へ移す
- `CLAUDE.md` — Claude Code 向けの進め方の約束（PR・Issue の作り方、ラベル、バグを見つけたときの手順など）。プロジェクトに合わせて書き足す
- `.github/workflows/sync-labels.yml` — `labels.json` の内容を GitHub のラベルへ自動反映
- `.github/dependabot.yml` — 依存関係の自動更新
- `.github/ISSUE_TEMPLATE/` — バグ報告・機能要望・調査用の Issue テンプレート
- `.github/pull_request_template.md` — PR テンプレート
- `labels.json` — プロジェクト共通のラベル定義（種別・PR・Issue・タスク分解・コメント・ステータスの6カテゴリ）

## 使い方
1. このリポジトリを新規プロジェクトの雛形として使用する
2. `sum.js` / `sum.test.js` などのサンプルコードを実際の実装・テストに置き換える
3. 雛形部分（CI/CD・Issue/PR テンプレート・Dependabot 設定）はそのまま利用する
   - 本番に出す処理は `deploy.sh` に書く（実行できるようにしておく: `chmod +x deploy.sh`）。ないあいだは、承認しても何もしない
   - 「[本番の前に承認を待つ](#本番の前に承認を待つ)」と「[GitHub Projects で進み具合を管理する](#github-projects-で進み具合を管理する)」の設定を行う
   - `CLAUDE.md` の「構成」「マイルストーン」などを、プロジェクトに合わせて書く
4. GitHub の「Actions」タブから `Sync Labels` ワークフローを手動実行（Run workflow）し、`labels.json` に定義されたラベルを反映する
   - 以後は `labels.json` を編集して `main` に push すると自動で同期される

## ラベル管理について
GitHubのラベルは、リポジトリを作るたびに手動で作り直すのが面倒になりがちです。このテンプレートでは [labels-config](https://zenn.dev/ait/articles/zenn-labels-config-guide) というツールを使い、`labels.json` に定義した内容をコマンド一つ（またはpush時に自動で）GitHubへ反映できるようにしています。

まだ導入したことがない方向けに、最低限の使い方を紹介します。

### 導入手順
```bash
# 1. GitHub CLI のセットアップ（未導入の場合）
gh auth login

# 2. labels-config のインストール
npm install -g @asagiri-design/labels-config

# 3. 動作確認
labels-config --version
```

### ローカルで手動同期する場合
```bash
# プロジェクトのディレクトリで実行
labels-config sync --owner {owner} --repo {repo} --file labels.json --dry-run   # 事前確認
labels-config sync --owner {owner} --repo {repo} --file labels.json --delete-extra   # 反映
```

`--owner`・`--repo`を毎回指定するのが面倒な場合は、記事内で紹介されているシェル関数（`labels-sync`）を導入すると、現在のディレクトリから自動でリポジトリ情報を取得してくれます。詳しくは[記事](https://zenn.dev/ait/articles/zenn-labels-config-guide) を参照してください。

### GitHub Actionsで自動化する場合
本テンプレートには `.github/workflows/sync-labels.yml` を同梱済みです。`labels.json` を編集して `main` に push するだけで、GitHub Actions が自動でラベルを同期します。初回のみ、Actionsタブから `Sync Labels` を手動実行してください（詳細は「[使い方](#使い方)」参照）。

## ブランチ保護ルールの設定
このテンプレートから作成したリポジトリでは、CI/CD やレビューを確実に機能させるため、`main` ブランチに保護ルールを設定することを推奨します。
Template repository は Branch protection rules をコピーしないため、新規プロジェクトを作るたびに以下の手動設定が必要です。

1. リポジトリの **Settings** → **Branches** を開く
2. **Add branch protection rule** をクリック
3. Branch name pattern に `main` を入力
4. 以下の項目にチェック
   - Require a pull request before merging
   - Require approvals（1名以上）
   - Require status checks to pass before merging（`ci.yml` のチェックを指定。事前に一度 CI を実行しておく必要があります）
   - Do not allow bypassing the above settings
5. **Create** をクリックして保存

### （参考）GitHub CLI で設定する場合
```bash
gh api --method PUT /repos/{owner}/{repo}/branches/main/protection \
  -f "required_pull_request_reviews[required_approving_review_count]=1" \
  -F "enforce_admins=true" \
  -f "restrictions=null"
```

## 本番の前に承認を待つ
`cd.yml` は、main にマージしてもすぐには本番に出さず、利用者の承認を待ちます。バグのあるコードをマージしても、承認しなければ本番には届きません。

### 流れ
1. PR をマージする → マージした PR に、承認・却下の画面のリンクが自動でコメントされる
2. 動きを確かめて、Actions の画面の「Review deployments」→「production」→「Approve and deploy」（取りやめるなら「Reject」）
3. 本番に出る（`deploy.sh`）→ 承認したコミットに `release-<番号>` のタグが付く → GitHub Projects の列が動く

### 最初に行う設定
1. **Settings** → **Environments** → **New environment** で `production` を作る（すでにあれば、それを開く）
2. **Required reviewers** にチェックを入れ、承認する人を追加して **Save protection rules**
   - 一人で開発しているなら **Prevent self-review** はオフのまま（オンにすると自分で承認できない）
   - この設定がないと、承認を待たずにそのまま本番に出るので注意

### スマホで承認・却下する
GitHub Mobile アプリを入れ、設定のプッシュ通知で「Deployment reviews」をオンにすると、承認待ちになったときに通知が届き、そこから承認・却下できます。

### バグを見つけたとき
承認前なら「Reject」して本番に出さず、直す PR を作るか、マージした PR のページの **Revert** ボタンで取り消します。直すまでは、ほかの変更も承認しないでください（main に残ったバグも一緒に出てしまうため）。

## GitHub Projects で進み具合を管理する
Issue と PR を、カンバン（ボード）で管理します。列は次の 4 つにし、ほとんど自動で動くようにします。

| 列 | 意味 | 動き方 |
|---|---|---|
| 未着手 | まだ始めていない | 開いた Issue と PR が自動で入る |
| 作業中 | 作業している | 手で移す |
| 承認待ち | マージ済み。本番への反映の承認を待っている | PR をマージすると自動で移る |
| 本番反映済み | 本番に反映された（Issue は完了） | 承認されると `cd.yml` が自動で移す。閉じた Issue も移る |

### 作り方
1. 「Projects」タブ → 「New project」→「Board」で作る。名前はリポジトリ名にする（違う名前にするときは、Actions の Variables に `PROJECT_TITLE` を作って、その名前を入れる）
   - 「Import items from repository」で、このリポジトリを選ぶと、開いた Issue と PR が自動で入るようになる
2. 列（Status）の名前を、上の表の 4 つに変える（「承認待ち」「本番反映済み」は、スクリプトが名前で探すので**このとおり**にする）
3. 右上の **Workflows** で、次のように設定する

| Workflow | 設定 |
|---|---|
| Auto-add to project | オン。条件は `is:issue,pr is:open` |
| Item added to project | オン。Status を「未着手」 |
| Pull request merged | オン。Status を「**承認待ち**」 |
| Item closed | オン。対象は **Issue だけ**、Status を「本番反映済み」（PR も対象にすると、マージした時点で本番反映済みに移ってしまう） |
| Auto-archive items | オン。条件は `is:issue,pr is:closed,merged updated:<@today-4w`（閉じた・マージしてから 4 週間たったものを自動でしまう。しまったものは右上の … → Archived items で見られる） |

4. プロジェクトを読み書きできるトークンを作り、Secrets に登録する（`cd.yml` の `project` ジョブが使う）
   - 右上のアイコン → Settings → Developer settings → Personal access tokens → **Tokens (classic)** → Generate new token (classic)
   - Scopes は **`project` だけ**にチェックし、期限を決めて作る（期限は `CLAUDE.md` の「期限のあるもの」に書いておく）
   - リポジトリの Settings → Secrets and variables → Actions → New repository secret で、名前 `PROJECT_TOKEN` として登録する
   - 登録しないあいだは、列が自動で移らないだけで、本番への反映は止まらない
   - プロジェクトの持ち主がリポジトリの持ち主と違う（例: リポジトリは組織、プロジェクトは個人）ときは、Actions の Variables に `PROJECT_OWNER` を作って、プロジェクトの持ち主の名前を入れる

### あると便利なビュー
| ビュー | 作り方 | 使いみち |
|---|---|---|
| ボード | Board（Status の列） | 進み具合を見る。View → Fields で Labels・Milestone・Type をカードに出すと分かりやすい |
| 一覧 | Table | View → Fields で Labels・Milestone・Type を表示。Sort by を Updated にすると最近動いたものが上に来る |
| マイルストーン別 | Board で、View → **Swimlanes** を Milestone にする | 目標ごとに、どこまで進んだかを見る（Board の列の分け方は Status しか選べないので、Swimlanes を使う） |
| ロードマップ | Roadmap で、View → Dates に開始日・終了日を選ぶ | 予定を時間軸で見る（下の日付の項目が必要） |

### Issue の種類と日付
- Issue テンプレートには種類（Type）が付く（不具合の報告 → Bug、機能の要望 → Feature、調査・質問 → Task）
- 予定を管理したいときは、アカウント（または組織）の **Settings → Planning → Issue fields** で、種類 Date の項目「開始日」「終了日」を作る。Pin to issues で Task・Bug・Feature を選ぶと、Issue の画面に常に表示される
