# build_app

[nxnative](https://github.com/seedds/nxnative)（上流）を追跡して、自分のゲームの IPA をビルドし、
GitHub Actions の artifact に **最新コミットの成功ビルド 1 つだけ** を残すリポジトリ。

ワークフロー: [`.github/workflows/nxnative-ipa.yml`](.github/workflows/nxnative-ipa.yml)

## 動作

- 6 時間ごと（cron）と手動実行で `seedds/nxnative` の main の最新コミットを確認する
- そのコミットのビルド済み artifact（`nxnative-ipa-<コミット12桁>`）が**なければ**ビルド、あればスキップ
- ビルドが成功したら新しい artifact をアップロードし、その後で**古いビルドの artifact をすべて削除**する（常に 1 つだけ残る）
- ビルドが失敗しても何も削除しない。前の成功ビルドが残り、次回の実行で自動でリトライする

## セットアップ

> **このリポジトリは private にすることを推奨。**
> artifact にはゲームのデータが入る。public リポジトリの artifact は誰でもダウンロードできる。
> private にすると macOS ランナーの時間がプランの無料枠から消費されるが、上流に変更が
> ないときはビルド全体をスキップするので、実際にビルドが走るのは上流が動いたときだけ。

### 1. ゲームの置き場所（どちらか一方）

**方法 A — プライベートリポジトリ**（例: `keewai704/nxnative-game` のルートにゲームを置く）

- ゲームフォルダの内容: 本体＋アップデート＋DLC の NSP/XCI/NSZ、dump フォルダ、または `.nup` ファイル 1 個
- **Variables**: `GAME_REPO` = リポジトリ名（例: `keewai704/nxnative-game`）
- **Secrets**: `GAME_PAT` = そのリポジトリが読める fine-grained PAT（Contents: Read-only、対象はそのリポジトリのみ）

**方法 B — URL**（クラウドストレージ、自分のサーバー、private リポジトリの release asset など）

- **Secrets**: `GAME_URL` = ゲームフォルダの zip/tar、または `.nup` ファイル 1 個の URL

### 2. prod.keys（NSP/XCI/NSZ のときのみ。`.nup` や dump は不要）

- **Secrets**: `PROD_KEYS_B64` = base64 した prod.keys（macOS なら `base64 -i prod.keys | tr -d '\n' | pbcopy`）

### 3. 任意の Variables

| 変数 | 意味 |
|---|---|
| `TITLE_ID` | フォルダに複数のゲームがあるとき、ビルドするゲーム（16 桁 hex） |
| `ROMFS` | `bundle`（デフォルト: ゲームファイルを IPA に同梱）/ `separate`（IPA は小さくなり、ゲームファイルは `scripts/push_romfs.sh` で 1 回だけ携帯に転送） |
| `LANGUAGE` | 例: `ja`（空 = 英語。アプリの Documents/env.txt で後から変えられる） |

## 使い方

- **IPA のダウンロード**: Actions → nxnative IPA → 成功した実行 → Artifacts の `nxnative-ipa-xxxx`
  （中身: IPA ＋ `BUILD-INFO.txt`：ビルドした上流コミットと日時）
- AltStore / Sideloadly / LiveContainer で署名してインストール。**Increased Memory Limit を有効に**
- **手動ビルド**: Run workflow から。`ref` で上流のブランチ・タグ・SHA を指定できる。ゲームを差し替えたあとは `force` で再ビルド
- 失敗時は `build-logs-xxxx` artifact（7 日間）にビルドログが残る

## 制約・注意

- ビルドにはゲームサイズの約 2 倍＋10 GB のディスクが必要。ランナーの空きはビルドログの `df -h` で確認できる。大きいゲームは GitHub の macOS ランナーに収まらないことがある → `ROMFS=separate` かローカルでのビルドを
- 実行時にシェーダーを生成するゲーム（Super Mario Galaxy など、Mac 側で一度起動してキャプチャする必要があるもの）は、CI の仮想 Mac（Metal 4 GPU なし、nxnative issue #112）ではビルドできない
- 初回ビルドは 30〜60 分。以降は work フォルダのキャッシュ（上限 10 GB/リポジトリ）で速くなる
- Xcode 27.2（`xcode-27` イメージ）に固定。上流がツールチェイン要件を変えたらワークフローの更新が必要
- public リポジトリのままだと、60 日間コミットがないとき scheduled workflow が自動停止する → 何かコミットするか手動実行
