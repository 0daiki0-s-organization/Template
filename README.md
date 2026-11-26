# Template
新規プロジェクト用のテンプレートリポジトリ。Node.js + Jest によるテスト、GitHub Actions による CI/CD（本番の前に承認を待つ形）、Issue/PR テンプレート、ラベル、Dependabot、GitHub Projects の自動化、Claude Code 向けの約束（`CLAUDE.md`）をあらかじめ用意してある。

このテンプレートから新しいリポジトリを作ったら、まず「[新しいリポジトリを作ったら行う設定](#新しいリポジトリを作ったら行う設定)」を上から順に進める。

## 目次
- [手元で動かす](#手元で動かす)
- [新しいリポジトリを作ったら行う設定](#新しいリポジトリを作ったら行う設定)
- [ふだんの流れ（PR から本番まで）](#ふだんの流れpr-から本番まで)
- [バグを見つけたとき](#バグを見つけたとき)
- [ファイルの説明](#ファイルの説明)
- [ラベル管理について](#ラベル管理について)

## 手元で動かす
```bash
npm install
npm test
```

## 新しいリポジトリを作ったら行う設定
Template repository からリポジトリを作っても、**ファイル以外の設定（ブランチの保護・Environments・Secrets・ラベル・マイルストーン・プロジェクトなど）はコピーされない**。次の順に設定する。

- [ ] [1. テンプレートからリポジトリを作る](#1-テンプレートからリポジトリを作る)
- [ ] [2. サンプルコードと deploy.sh を用意する](#2-サンプルコードと-deploysh-を用意する)
- [ ] [3. ラベルを反映する](#3-ラベルを反映する)
- [ ] [4. マイルストーンを作る](#4-マイルストーンを作る)
- [ ] [5. main を守る（ルールセット）](#5-main-を守るルールセット)
- [ ] [6. 本番の前に承認を待つ（Environments）](#6-本番の前に承認を待つenvironments)
- [ ] [7. Secrets と Variables を登録する](#7-secrets-と-variables-を登録する)
- [ ] [8. GitHub Projects を作る](#8-github-projects-を作る)
- [ ] [9. Issue の日付の項目を作る（任意）](#9-issue-の日付の項目を作る任意)
- [ ] [10. セキュリティの設定](#10-セキュリティの設定)
- [ ] [11. スマホで承認・却下できるようにする](#11-スマホで承認却下できるようにする)
- [ ] [12. テンプレートと CLAUDE.md をプロジェクトに合わせる](#12-テンプレートと-claudemd-をプロジェクトに合わせる)

### 1. テンプレートからリポジトリを作る
1. このリポジトリの **Use this template** → **Create a new repository**
2. 持ち主（個人か組織）と名前を決めて作る

### 2. サンプルコードと deploy.sh を用意する
- `sum.js` / `sum.test.js` を、実際の実装とテストに置き換える
- `package.json` の名前（`name`）を変える
- 本番に出す処理を、リポジトリの直下の `deploy.sh` に書き、実行できるようにする
  ```bash
  chmod +x deploy.sh
  ```
  - `deploy.sh` がないあいだは、承認しても本番には何もしない（警告が出るだけ）
  - 本番に出すのに必要な鍵は、コードに書かず Secrets に入れる（[7.](#7-secrets-と-variables-を登録する)）
- CI で使う Node.js の版は `.github/workflows/ci.yml` の `node-version`（既定は 24）

### 3. ラベルを反映する
1. **Actions** タブ → **Sync Labels** → **Run workflow**
2. 「変更せずに、行う予定の操作だけを表示する」の**チェックを外して**実行する（チェックしたままだと試運転だけで、ラベルは変わらない）
3. 以後は、`labels.json` を書き換えて main にマージすると、自動で反映される

ラベルの使い分けは `CLAUDE.md` の「ラベル・マイルストーン・プロジェクト」に書いてある。

### 4. マイルストーンを作る
1. **Issues** タブ → **Milestones** → **New milestone**
2. プロジェクトの大きな目標ごとに作る（例: 「保守・運用」「○○の公開」）。期限（Due date）が決まっていれば入れる。期限を入れると、ロードマップのビューに目印が出る
3. 作ったマイルストーンの番号と意味を、`CLAUDE.md` の「マイルストーン」に書く（Claude Code が PR や Issue に付けるときに使う）

### 5. main を守る（ルールセット）
壊れた変更が main に入らないようにする。

1. **Settings** → **Rules** → **Rulesets** → **New ruleset** → **New branch ruleset**
2. 名前（例: `main`）を付け、**Enforcement status** を **Active** にする
3. **Target branches** → **Add target** → **Include default branch**
4. 次にチェックを入れる
   - **Restrict deletions**（main を消せないように）
   - **Block force pushes**（履歴を書き換えられないように）
   - **Require a pull request before merging**（PR を通さずに main に入れないように）
     - 一人で開発しているなら **Required approvals は 0** にする（1 以上にすると、自分の PR をマージできない）
   - **Require status checks to pass**（CI が通らないとマージできないように）
     - **Add checks** で `build`（`ci.yml` のジョブ名）を選ぶ。一度 PR を作って CI を動かしておくと、候補に出る
5. **Create** で保存する

### 6. 本番の前に承認を待つ（Environments）
`cd.yml` は、main にマージしてもすぐには本番に出さず、承認を待つ。バグのあるコードをマージしても、承認しなければ本番には届かない。

1. **Settings** → **Environments** → **New environment** で `production` を作る（すでにあれば、それを開く）
2. **Required reviewers** にチェックを入れ、承認する人を追加する
   - 一人で開発しているなら **Prevent self-review** はオフのまま（オンにすると自分で承認できない）
   - **Allow administrators to bypass configured protection rules** はオンのままでよい（管理者が急ぎのときに承認を飛ばせる。何もしなければ承認待ちで止まる）
3. **Deployment branches and tags** は **No restriction** のままでよい
4. **Save protection rules** で保存する

> この設定がないと、承認を待たずにそのまま本番に出るので注意。

### 7. Secrets と Variables を登録する
**Settings** → **Secrets and variables** → **Actions** で登録する。秘密の値は **Secrets**、秘密でない設定は **Variables** に入れる。

| 種類 | 名前 | 使うところ | 内容 |
|---|---|---|---|
| Secret | `PROD_API_KEY` | `cd.yml` の `deploy` | 本番に出すのに必要な鍵（例）。`deploy.sh` に合わせて名前を変え、`cd.yml` の `env` も直す |
| Secret | `PROJECT_TOKEN` | `cd.yml` の `project` | プロジェクトを読み書きできるトークン（作り方は [8.](#8-github-projects-を作る) の 4） |
| Variable | `PROJECT_TITLE` | `cd.yml` の `project` | プロジェクトの名前。リポジトリ名と同じなら不要 |
| Variable | `PROJECT_OWNER` | `cd.yml` の `project` | プロジェクトの持ち主。リポジトリの持ち主と同じなら不要（例: リポジトリは組織、プロジェクトは個人のとき） |

トークンなどの期限は、`CLAUDE.md` の「期限のあるもの」に書いておく。

### 8. GitHub Projects を作る
Issue と PR をカンバン（ボード）で管理する。列は次の 4 つにし、ほとんど自動で動くようにする。

| 列 | 意味 | 動き方 |
|---|---|---|
| 未着手 | まだ始めていない | 開いた Issue と PR が自動で入る |
| 作業中 | 作業している | 手で移す |
| 承認待ち | マージ済み。本番への反映の承認を待っている | PR をマージすると自動で移る |
| 本番反映済み | 本番に反映された（Issue は完了） | 承認されると `cd.yml` が自動で移す。閉じた Issue も移る |

1. **プロジェクトを作る**
   - 「Projects」タブ（または右上のアイコン → Your projects）→ **New project** → **Board**
   - 名前はリポジトリ名にする（違う名前にするときは、Variables の `PROJECT_TITLE` にその名前を入れる）
   - **Import items from repository** にチェックを入れ、このリポジトリを選ぶ（開いた Issue と PR が自動で入るようになる）
2. **列の名前を変える**
   - 列の名前の右の **…** → **Edit details** で、上の表の 4 つの名前と説明に変える（足りない列は右端の **＋** で作る）
   - 「承認待ち」「本番反映済み」は、スクリプトが名前で探すので**このとおり**にする
3. **自動で動く設定（Workflows）**
   - 右上の **Workflows** で、次のように設定する（それぞれ **Edit** → 設定 → **Save and turn on workflow**）

   | Workflow | 設定 |
   |---|---|
   | Auto-add to project | オン。条件は `is:issue,pr is:open` |
   | Item added to project | オン。Status を「未着手」 |
   | Pull request merged | オン。Status を「**承認待ち**」 |
   | Item closed | オン。対象は **Issue だけ**、Status を「本番反映済み」（PR も対象にすると、マージした時点で本番反映済みに移ってしまう） |
   | Auto-archive items | オン。条件は `is:issue,pr is:closed,merged updated:<@today-4w` |

   - Auto-archive items は、閉じた・マージしてから 4 週間たったものを自動でしまう。しまったものは右上の **…** → **Archived items** で見られ、**Restore** で戻せる
   - マージしてから 4 週間以上承認しないと、PR が「承認待ち」のまましまわれ、承認しても「本番反映済み」に移らない。そのときは Archived items から戻す
4. **トークンを作る**（承認後に「本番反映済み」へ移すのに使う）
   - 右上のアイコン → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)** → **Generate new token (classic)**
   - **Note** に用途（例: `<リポジトリ名> project`）、**Expiration** に期限（例: 1 年）、**Scopes** は **`project` だけ**にチェックして作る
   - 表示されたトークンをコピーし、[7.](#7-secrets-と-variables-を登録する) の `PROJECT_TOKEN` に登録する（この画面でしか見られない）
   - 登録しないあいだは、列が自動で移らないだけで、本番への反映は止まらない
5. **ビューを作る**（ビューの名前は自由に付けてよい）
   - 「＋ New view」で作り、ビューの名前は ▾ → **Rename view**、表示の設定は右上の **View** で変えて **Save**

   | ビュー | 作り方 | 使いみち |
   |---|---|---|
   | ボード | Board（Status の列）。View → Fields で Labels・Milestone・Type をカードに出す | 進み具合を見る |
   | 一覧 | Table。View → Fields で Labels・Milestone・Type、Sort by を Updated | 種類・目標を並べて見る。最近動いたものが上に来る |
   | マイルストーン別 | Board で、View → **Swimlanes** を Milestone | 目標ごとに、どこまで進んだかを見る（Board の列の分け方は Status しか選べないので、Swimlanes を使う） |
   | ロードマップ | Roadmap で、View → Dates に開始日・終了日（または Milestone） | 予定を時間軸で見る（[9.](#9-issue-の日付の項目を作る任意) の日付の項目が必要） |

   - マイルストーン別で、絞り込みの欄に `-status:本番反映済み` と入れると、目標ごとの残りの作業だけを見られる
   - 「作業中」の列の **…** → **Set limit** で上限（例: 3）を決めると、抱えすぎたときに知らせてくれる
6. **Insights（任意）**: 右上の **Insights** で、Status ごとの件数や、完了した数の推移のグラフを作れる

### 9. Issue の日付の項目を作る（任意）
予定（いつ始めて、いつまでに終わらせるか）を管理したいときに作る。

1. アカウント（組織のリポジトリなら組織）の **Settings** → **Planning** → **Issue fields** → 新しい項目
2. 名前 `開始日`、種類 **Date**、**Permissions** は Restricted のままでよい
3. **Pin to issues** の鉛筆のマークで、Task・Bug・Feature（必要なら種類のない Issue も）を選ぶ（選ばないと、日付が入っていない Issue の画面に項目が出ない）
4. **Create** で作る。同じように `終了日` も作る
5. プロジェクトの表のビューで **＋ Add field** から追加すると、日付の列が出る

Issue の種類（Type）は Task・Bug・Feature の 3 つ。Issue テンプレートから作ると自動で付く（不具合の報告 → Bug、機能の要望 → Feature、調査・質問 → Task）。

### 10. セキュリティの設定
1. **Settings** → **Advanced Security**（または **Code security**）で、次をオンにする
   - **Dependabot alerts**（脆弱性のある依存パッケージを知らせる）
   - **Dependabot security updates**（脆弱性を直す PR を自動で作る）
   - **Secret scanning** と **Push protection**（トークンなどの秘密の値をコミットしそうになったら止める）
2. 依存パッケージの定期的な更新は `.github/dependabot.yml`（既定は npm を毎日確認）
3. GitHub アカウントの **2 段階認証**をオンにしておく（Settings → Password and authentication）

### 11. スマホで承認・却下できるようにする
1. スマホに **GitHub Mobile** アプリを入れてログインする
2. アプリの設定 → 通知で、プッシュ通知の **Deployment reviews**（承認待ち）をオンにする
3. 承認待ちになると通知が届く。開いて **Review deployments** → **production** → **Approve** または **Reject**

### 12. テンプレートと CLAUDE.md をプロジェクトに合わせる
- **Issue テンプレート**（`.github/ISSUE_TEMPLATE/`）: 不具合の報告の「どこで起きましたか」の選択肢（Web・iOS・Android など）を、プロジェクトに合わせて直す。問い合わせ窓口があれば `config.yml` の `contact_links` に書く
- **PR テンプレート**（`.github/pull_request_template.md`）: そのまま使える。Issue に対応する PR は「関連タスク」に `Closes #番号` と書くと、マージしたときに Issue が自動で閉じる
- **`CLAUDE.md`**: Claude Code が新しいセッションの最初に読む約束。「構成」「マイルストーン」「気をつけること」「期限のあるもの」をプロジェクトに合わせて書く
- **README**: このファイルを、プロジェクトの説明に書き換える（設定の手順は、残しておくと後で役に立つ）

## ふだんの流れ（PR から本番まで）
1. 作業用のブランチで変更し、PR を**ドラフト**で作る（ドラフトは「作業中なのでマージしない」という目印）。ラベルとマイルストーンを付ける
2. 作業が終わり、CI（`ci.yml`）がすべて成功したら **Ready for review** にする。Ready for review の PR だけマージする
3. マージすると、PR に承認・却下の画面のリンクがコメントされ、本番の前で止まる。プロジェクトでは「承認待ち」に移る
4. 動きを確かめて、Actions の画面（またはスマホの通知）の **Review deployments** → **production** → **Approve and deploy**
5. 本番に出る（`deploy.sh`）→ 承認したコミットに `release-<番号>` のタグが付く → プロジェクトで「本番反映済み」に移る

本番に出した版は、リポジトリの **Tags** の `release-<番号>` で確かめられる。

## バグを見つけたとき
承認しない限り本番には出ないが、直すまで main にはバグが残る。その間に承認すると、ほかの変更と一緒にバグも本番に出るので、直すまでは承認しない。

| 状況 | すること |
|---|---|
| マージ後・承認前に見つけた | 1. その実行を **Reject** する（本番に出さない）<br>2. すぐ直せるなら、直す PR を作る<br>3. 時間がかかりそうなら、マージした PR のページの **Revert** ボタンで取り消して main を戻してから、落ち着いて直す |
| 承認したあと、本番で見つけた | 1. 本番を前の版に戻す（戻し方はプロジェクトごとに決めて `CLAUDE.md` に書く）<br>2. Revert か、直す PR を作る |

Revert の PR も直す PR も、ふだんと同じ流れ（ドラフト → Ready for review → マージ → 承認）で出す。

## ファイルの説明
### サンプルコード（実装に合わせて削除・置き換える）
- `sum.js` — Jest の動作確認用サンプル実装
- `sum.test.js` — 上記に対応するサンプルテスト

### 雛形（そのまま利用する土台）
- `package.json` — npm scripts・依存関係の定義
- `.github/workflows/ci.yml` — main への push・PR のときにテストを実行（Node.js 24）
- `.github/workflows/cd.yml` — main へのマージ後、承認を待ってから本番に出す（`deploy.sh`）。承認した版に `release-<番号>` のタグを付け、GitHub Projects の列も動かす
- `.github/scripts/move-released-to-done.sh` — 承認後に、GitHub Projects の「承認待ち」の PR のうち今回出したものを「本番反映済み」へ移す（個人・組織どちらのプロジェクトにも対応）
- `.github/workflows/sync-labels.yml` — `labels.json` の内容を GitHub のラベルへ反映
- `.github/dependabot.yml` — 依存パッケージの自動更新
- `.github/ISSUE_TEMPLATE/` — 不具合の報告・機能の要望・調査の Issue テンプレート（種類 Type も自動で付く）
- `.github/pull_request_template.md` — PR テンプレート
- `labels.json` — 共通のラベル定義（種別・PR・Issue・タスク分解・コメント・ステータス・Dependabot）
- `CLAUDE.md` — Claude Code 向けの進め方の約束（PR・Issue の作り方、ラベル、バグを見つけたときの手順など）

## ラベル管理について
GitHub のラベルは、リポジトリを作るたびに手動で作り直すのが面倒になりがち。このテンプレートでは、`labels.json` に定義した内容を GitHub Actions（`sync-labels.yml`）で自動的に反映する（[3.](#3-ラベルを反映する)）。

手元から反映したい場合は、[labels-config](https://zenn.dev/ait/articles/zenn-labels-config-guide) というツールも使える。

```bash
# 1. GitHub CLI のセットアップ（未導入の場合）
gh auth login

# 2. labels-config のインストール
npm install -g @asagiri-design/labels-config

# 3. 反映（--dry-run で事前確認、--delete-extra で labels.json にないラベルを削除）
labels-config sync --owner {owner} --repo {repo} --file labels.json --dry-run
labels-config sync --owner {owner} --repo {repo} --file labels.json --delete-extra
```

`--owner`・`--repo` を毎回指定するのが面倒な場合は、[記事](https://zenn.dev/ait/articles/zenn-labels-config-guide) で紹介されているシェル関数（`labels-sync`）を使うと、今のディレクトリから自動でリポジトリの情報を取ってくれる。
