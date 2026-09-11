# FUIDE Git

[FUIDE](https://github.com/kobago/fuide) (Sci-Fi / FUI デザインの egui 部品ライブラリ `fuide`) の上に載せた Git クライアント (macOS)。

```
apps/git/        FUIDE Git — 読み書きとも `git` CLI、hunk 単位のステージ、コミットグラフ、fetch / push のストリーミング出力
assets/icons/    .app のアイコン (SVG)
scripts/         release.sh (.app / DMG)
```

`fuide` は git 依存 (`Cargo.toml` の `[workspace.dependencies]`)。`fuide` クレート自体を隣の `../fuide` で直しながら動かすときは `Cargo.toml` 末尾のコメントの `[patch]` を外す。

共通の仕組み (設定ウィンドウ、MCP エージェント、テストの決めごと、再描画レート) は [kobago/fuide](https://github.com/kobago/fuide) の README を参照。

## FUIDE Git

```sh
cargo run -p fuide-git                 # 最近開いたリポジトリ (無ければカレントディレクトリ)
cargo run -p fuide-git -- ~/src/repo   # リポジトリを指定して開く
```

Git クライアント ([#4](https://github.com/kobago/fuide/issues/4))。**libgit2 / gitoxide は使わず、読み書きともに `git` CLI だけ**を呼ぶ (`apps/git/src/git.rs`)。読み取りは `status --porcelain=v2 -z` / `log` / `for-each-ref` / `diff` / `show` を別スレッドで実行して 1 メッセージで返す。変更系 (`add` / `restore` / `commit` / `switch` / `fetch` / `pull` / `push` / `apply`) はすべて brew と同じストリーミング runner を通り、出力が 1 行ずつログに流れ、完了後にリポジトリを読み直す。同時実行は 1 つ。fetch / push は git 自身の credential helper に任せる (`GIT_TERMINAL_PROMPT=0` なので対話は起きず、失敗は ERROR カード)。

- **左上: REPOSITORY** — 名前、パス、ブランチ、upstream、ahead / behind、OPEN (パス入力ダイアログ、Tab 補完) / FETCH、最近開いたリポジトリ (`~/Library/Application Support/FUIDE/git-recent.conf`)。Finder や ffm からディレクトリをドロップしても開く
- **左下: BRANCHES** — NEW BRANCH (`switch -c`)、LOCAL / REMOTES / TAGS の一覧 (現在のブランチが点灯、右に upstream)。**ダブルクリックで切替** (`switch`。リモートは同名のローカルを作って追跡、タグは detach)
- **中央: CHANGES ビュー** (Cmd+1) — UNSTAGED / STAGED の 2 表 (ST / PATH)。行クリックで下に diff、**ダブルクリックか Space でステージ / アンステージ**、STAGE ALL (`add -A`、Cmd+A) / UNSTAGE ALL (`reset`) / DISCARD (`restore` または untracked は `clean -f`、危険色の確認ダイアログでエージェントは人間留保)。下の DIFF は行番号 (旧 / 新)、追加 = 緑、削除 = 危険色、hunk 行に **STAGE HUNK / UNSTAGE HUNK** (`git apply --cached [-R]` にその hunk だけの patch を流す)
- **中央: HISTORY ビュー** (Cmd+2) — `log --all --topo-order` の直近 500 件 (グラフ / HASH / SUBJECT / AUTHOR / WHEN、装飾付きは accent)。先頭列が**コミットグラフ**: `git::graph` が親リストからレーンを割り当て (第 1 親は同じレーンを引き継ぎ、第 2 親以降は待っているレーンか空きレーン、分岐線は分岐点の行まで伸びる gitk 流)、表の行ごとに線 (グロー付き) とノード (輪、HEAD は塗り + 脈動) を描く (`table::table_decorated` の行フック)。レーン色は accent / ok / warn / accent_dim の循環、8 レーンまで表示。行を選ぶと右にコミット詳細、その変更ファイルをクリックすると下に diff (`show <hash> -- path`)
- **右: COMMIT** (CHANGES ビュー) — メッセージ欄と COMMIT (staged があり、メッセージが空でないとき。`commit -F -` で stdin から渡す。Cmd+Enter は欄にフォーカスがあっても効く)。HISTORY ビューでは選択コミットの件名 / 本文 / hash / author / date / parents / refs、COPY HASH、ファイル一覧
- **ツールバー**: 再読込 (Cmd+R)、ビュー切替、PULL (`--ff-only`、behind 数付き) / PUSH (ahead 数付き。upstream が無ければ `-u origin <branch>`)
- **下: GIT OUTPUT** — コマンドの標準出力 / 標準エラー (`error` / `fatal` = 危険色、`warning` / `hint` = 注意色)。帯のドラッグで高さ変更、チップのクリックで開閉

| 操作 | キー |
|---|---|
| 選択移動 | ↑↓ (フォーカス中の表: UNSTAGED / STAGED / HISTORY) |
| ステージ / アンステージ | Space または Enter (選択行)、ダブルクリック |
| 全部ステージ | Cmd+A |
| コミット | Cmd+Enter |
| リポジトリを開く / 再読込 | Cmd+O / Cmd+R |
| ビュー | Cmd+1 (CHANGES) / Cmd+2 (HISTORY) |
| 設定 / 終了 | Cmd+, / Cmd+W |

まだ無いもの ([kobago/fuide#7](https://github.com/kobago/fuide/issues/7)): reset / force push / branch -D、マージ競合の解決、500 件より古いログの段階読み込み、MCP の Git 専用ツール。

撮影フック: `FUIDE_DEV_DIALOG=diff|history|discard|open|branch|error|success`。テスト (`cargo test -p fuide-git`) は一時ディレクトリに `git init` した実リポジトリで回る (ネット不要。`GIT_CONFIG_GLOBAL=/dev/null` で署名などの個人設定を外す)。

## 配布 (.app / DMG、Apple Silicon)

```sh
cargo install cargo-bundle          # 初回のみ
./scripts/release.sh                # dist/FUIDE Git.{app,dmg}
```

- `cargo bundle --format osx` で `.app`（`Info.plist`、`assets/icons/*.svg` から `.icns`）→ `codesign`（既定は ad-hoc）→ `hdiutil` で `/Applications` へのリンク入り DMG
- バンドル設定は各 `apps/*/Cargo.toml` の `[package.metadata.bundle]`（識別子 `fuide.file-manager` / `fuide.brew`、最小 macOS 13）。`icon` のパスは cargo-bundle を実行したディレクトリ基準なので、スクリプトはワークスペース root で実行する
- Spotlight から起動するには DMG を開いて `.app` を `/Applications` にドラッグ（インデックスに数十秒。急ぐなら `mdimport /Applications/FUI\ Brew.app`）
- **他の Mac に配る場合**: Developer ID で署名・公証していないので、受け取った側は初回だけ右クリック → 開く、または `xattr -d com.apple.quarantine "/Applications/FUIDE Brew.app"` が必要。Developer ID を取得したら `SIGN_IDENTITY="Developer ID Application: ..." ./scripts/release.sh` で署名し、`xcrun notarytool submit dist/*.dmg --wait` → `xcrun stapler staple` で公証
- FUIDE Brew は launchd 起動の最小 `PATH` でも動くよう `brew` を `/opt/homebrew/bin` → `/usr/local/bin` → `PATH` の順で探す
