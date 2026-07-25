# Git 使い方 備忘録

## 1. プロジェクトの初期化

### パターンA: git clone（リモートからプロジェクトを丸ごとダウンロード）
* **用途:** 空のフォルダから始める場合、リモート（GitHub等）にあるプロジェクトをPCに丸ごとダウンロードする。
```bash
git clone <URL>
```

### パターンB: git init（PCにすでにあるプロジェクトを後からGitHubに登録する）
* **用途:** すでにローカルに存在するプロジェクトを、途中からGit管理下に置く場合に使う。
* （**Unityの場合**）Unity Hubでプロジェクト作成後、エディタを一度閉じてから該当フォルダで実行する。

```bash
cd <プロジェクトフォルダのパス>
git init
```
1. `git init`：該当フォルダをGit管理下に置く。

### .gitignore の取得・配置
* （**Unity等の場合**）不要なキャッシュ等（Library, Tempフォルダなど）を除外するための `.gitignore` を取得・配置する。
```bash
curl -o .gitignore https://raw.githubusercontent.com/github/gitignore/main/Unity.gitignore
```

### 初回コミット
```bash
git status
git add .
git commit -m "Initial commit"
```
2. `git status`：状態確認
3. `git add .`：全ファイルをステージングエリアに追加
4. `git commit -m "Initial commit"`：初回コミット（ローカルに保存）

### リモートリポジトリとの紐付けと初回送信
* GitHub上で空のリポジトリ（READMEなし）を作成後、以下で紐付けと初回の送信を行う。
```bash
git remote add origin <GitHubのURL>
git branch -M main
git push -u origin main
```
5. `git remote add origin <GitHubのURL>`：リモートリポジトリ（GitHub）のアドレスを登録・紐付け
6. `git branch -M main`：デフォルトブランチ名を「main」に変更
7. `git push -u origin main`：リモートへ初回送信（上流ブランチ設定）

## 2. ブランチ操作

### git switch（ブランチの切り替え）
* **git switch とは:** 作業するブランチ（枝）を切り替えるコマンド。
* 最初に本番の枝（main）へ切り替え、その後、自分専用の作業用の枝（feature/mypage）を作成して切り替える。
```bash
git switch main
git switch -c feature/mypage
```

### git branch（ブランチ一覧の確認）
* **git branch とは:** 現在ローカルに存在するブランチの一覧を確認するコマンド。今どのブランチにいるかは `*` マークで示される。
```bash
git branch
```
* （リモートにあるブランチも含めて確認したい場合）
```bash
git branch -a
```

### git pull（リモートの最新の更新を取り込む）
* **git pull とは:** リモート（GitHub）にある他のメンバーの最新の更新を、自分のPCに取り込むコマンド。
* ※ 通常は本番の枝 main に切り替えてから実行する。
```bash
git switch main
git pull
```

### 逆マージ（本番の変更を作業ブランチに同期）
* **逆マージとは:** 本番（main）の最新の変更を、自分の作業用ブランチ（feature/mypage）に取り込んで同期させる操作。
* ※ チーム開発では、他の人の変更との競合を事前に防ぐため、「自分がプルリクエストを出す直前」に頻繁に行われる。
```bash
git switch feature/mypage
git merge main
```

## 3. 基本操作 (status / add / commit / push)

### git status（変更状況の確認）
* **git status とは:** 現在の変更状況（どのファイルが変更・追加・削除されたか）を確認するコマンド。
* ※ add や commit を行う前の確認として頻繁に使用する。
```bash
git status
```

### git add（ステージングエリアへの追加）
* **git add とは:** 変更したファイルをコミット対象（ステージングエリア）として選択するコマンド。
* 「.」をつけることで、変更があったすべてのファイルをまとめて選択できる。
```bash
git add .
```
* （特定のファイルだけを選択したい場合：ファイル名やフォルダ名を直接指定する）
```bash
git add <ファイルパス>
```

### git commit（ローカルへの記録）
* **git commit とは:** 選択したファイルの変更を、メッセージをつけて自分のPC（ローカル）に歴史として記録するコマンド。
* 「`--amend -m "メッセージ"`」で直前のコミットメッセージの修正も可能。
```bash
git commit -m "マイページのレイアウトを作成"
```

### git push（リモートへのアップロード）
* **git push とは:** 自分のPCで記録したブランチを、リモート（origin）にそのままアップロードするコマンド。
* ※ `-u` をつけてプッシュすると、次回から同じブランチ（feature/mypage）にいれば単に「`git push`」だけでアップロードできるようになる。
```bash
git push -u origin feature/mypage
```
* （2回目以降の簡略化コマンド）
```bash
git push
```

## 4. 変更履歴・差分の確認 (log / diff)

### git log（コミット履歴の確認）
* **git log とは:** これまでのコミット履歴（誰が・いつ・どんなメッセージで記録したか）を確認するコマンド。
```bash
git log
```
* （1コミット1行で簡潔に表示したい場合。全体の流れを素早く把握できるため実務でもよく使う）
```bash
git log --oneline
```

### git diff（変更差分の確認）
* **git diff とは:** ファイルの中で「具体的に何行目のどこが変更されたか」を確認するコマンド。`git add` する前の確認に使う。
```bash
git diff
```
* （`git add` した後、まだコミットしていない変更内容を確認したい場合）
```bash
git diff --staged
```

## 5. プルリクエストとマージ

### プルリクエスト（Web上での操作）
* pushした自分の枝（feature/mypage）を本番の枝（main）に合流させてよいかチームに確認し、承認されたらGitHub上でマージする。

### コンフリクト（競合）が発生した場合の基本対応

* **コンフリクトとは**
  * 同じファイルの同じ箇所を複数人が別々に変更し、`git merge` や `git pull` の際にGitがどちらを採用すべきか自動判断できなくなった状態。`CONFLICT` という文字がターミナルに表示されたら発生の合図。

* **ファイルに挿入される記号の読み方**
  * 該当ファイルを開くと、以下のような記号が自動で挿入されている。
```
<<<<<<< HEAD
Hello World
=======
Hello Git
>>>>>>> feature/mypage
```
  * **`<<<<<<< HEAD` 〜 `=======`**：今、自分がいるブランチ側の内容
  * **`=======` 〜 `>>>>>>> ブランチ名`**：取り込もうとした相手側の内容

* **対応の基本手順**
  1. ファイルを開き、`<<<<<<<` 〜 `>>>>>>>` の記号を目印に「残したい内容」だけを選んで書き直す（記号自体は必ず削除する）。
  2. `git add <ファイル名>` でステージングする。
  3. `git commit -m "コンフリクトを解消"` でマージを完了させる。
```bash
git add <コンフリクトを解消したファイル>
git commit -m "コンフリクトを解消"
```

* **困ったときは中断できる**
  * 途中で分からなくなったら、以下のコマンドでマージ前の状態に戻せる。
```bash
git merge --abort
```

### ローカルマージ（1人練習用）
* **ローカルマージとは:** （1人練習用）GitHubを使用せず、自分のPC（ローカル）だけで本番の枝（main）に合流させる手順。
```bash
git switch main
git merge feature/mypage
git push origin main
```
1. `git switch main`：本番用の枝（main）に移動する
2. `git merge feature/mypage`：自分の作業用ブランチ（feature/mypage）をマージする
3. `git push origin main`：合流させた最新のローカルmainを、リモートのmainにアップロードする（git pushだけでも動く）