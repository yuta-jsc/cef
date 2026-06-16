# RWSB スキーム移行 運用手順書

この手順書は、`chrome://` を `rwsb://` に移行した変更を **cef.git 側で管理し、別ブランチ/次バージョンへ再適用**するための実運用手順です。

## 1. この変更で管理するもの

### 1.1 主要パッチ

- `patch/patches/chrome_ui_scheme_rwsb.patch`
  - WebUI の正規スキームを `rwsb://`（および `rwsb-untrusted://`）へ移行。
- `patch/patches/chrome_ui_scheme_runtime_compat.patch`
  - 既存の `chrome://` / `chrome-untrusted://` 文字列が残っていても、実行時に `rwsb://` / `rwsb-untrusted://` へ変換して吸収（CSP と WebUI テキスト系リソースの双方）。

### 1.2 patch.cfg 登録

`patch/patch.cfg` に以下が入っていること:

- `chrome_ui_scheme_rwsb`
- `chrome_ui_scheme_runtime_compat`

## 2. 現在ブランチでの確定手順（cef.git 管理）

`D:\cef_build_env\chromium_git\cef` で実行:

```bat
git status --short
```

必要な変更をステージ:

```bat
git add ^
  patch/patch.cfg ^
  patch/patches/chrome_ui_scheme_rwsb.patch ^
  patch/patches/chrome_ui_scheme_runtime_compat.patch ^
  patch/patches/chrome_browser_browser.patch ^
  patch/patches/chrome_browser_webui_license.patch ^
  patch/patches/chrome_browser_webui_version.patch ^
  libcef/browser/chrome/chrome_browser_host_impl.cc ^
  libcef/common/net/url_util.h ^
  tests/cefclient/browser/client_handler.cc ^
  tests/ceftests/webui_unittest.cc
```

コミット:

```bat
git commit -m "Add rwsb WebUI runtime compatibility patches"
```

## 3. 別ブランチへ同じ修正を入れる手順

### 3.1 cef.git のブランチ移動

```bat
cd /d D:\cef_build_env\chromium_git\cef
git switch <target-branch>
git cherry-pick <上記コミットSHA>
```

### 3.2 Chromium 側 patch の再整合（重要）

```bat
cd /d D:\cef_build_env\chromium_git\chromium\src\cef\tools
patch_updater.bat --resave --patch chrome_ui_scheme_rwsb
patch_updater.bat --resave --patch chrome_ui_scheme_runtime_compat
```

競合が出る場合:

1. 失敗したファイルを手で解消  
2. `patch_updater.bat --resave --patch <patch_name>` を再実行

### 3.3 パッチ一括適用確認

```bat
cd /d D:\cef_build_env\chromium_git\chromium\src\cef
tools\patch.bat
```

## 4. ビルド（通常運用コマンド）

### 4.1 実行前チェック（`chromium\src` をクリーン化）

```bat
cd /d D:\cef_build_env\chromium_git\chromium\src
git status --short
```

`git status --short` に出力がある状態で `automate-git.py` を実行すると、
patch 適用や `gclient_hook.py` で失敗することがあります。

未コミット変更がある場合は、先に退避してから実行します:

```bat
git stash push -u -m "wip-before-automate-rwsb"
```

退避した変更を戻す場合:

```bat
git stash list
git stash pop
```

### 4.2 推奨コマンド（`--no-cef-update` は付けない）

```bat
cd /d D:\cef_build_env\chromium_git

python3 automate-git.py ^
 --download-dir=D:\cef_build_env\chromium_git ^
 --depot-tools-dir=D:\cef_build_env\depot_tools ^
 --url=git@github.com:yuta-jsc/cef.git ^
 --branch=7778 ^
 --checkout=rwsb/schema_change ^
 --force-build --x64-build --build-target=cefclient
```

`--no-cef-update` を付けると CEF の `fetch/checkout` をスキップするため、既存 `chromium\src\cef` が古い場合に
`D:\cef_build_env\chromium_git\cef` の変更が反映されないことがあります。

`--no-cef-update` を使うのは、`chromium\src\cef` を別手順で同期済みであることを確認できる場合だけにしてください。

`--branch` と `--checkout` の役割は別です:

1. `--branch=7778`: 互換系統（Chromium対応ライン）を指定
2. `--checkout=...`: 実際に使う CEF の参照（ブランチ/コミット）を指定

独自ブランチ `rwsb/schema_change` を使う例:

```bat
python3 automate-git.py ^
 --download-dir=D:\cef_build_env\chromium_git ^
 --depot-tools-dir=D:\cef_build_env\depot_tools ^
 --url=git@github.com:yuta-jsc/cef.git ^
 --branch=7778 ^
 --checkout=origin/rwsb/schema_change ^
 --force-cef-update ^
 --force-build --x64-build --build-target=cefclient
```

再現性を固定したい場合は `--checkout=<コミットSHA>` を指定します。

`patch/patch.cfg` や `patch/patches/*.patch` を更新した直後は、`chromium\src\cef` 側が古いまま残るのを避けるため、
最初の1回は `--force-cef-update` を付ける運用を推奨します。

### 4.3 パッチ適用エラー時の復旧（`chrome_browser_webui_license` など）

`gclient_hook.py` 実行時に「patch failed to apply」が出た場合、`chromium\src\cef` と
`D:\cef_build_env\chromium_git\cef` の内容不一致、または `chromium\src` の未退避変更が原因であることが多いです。

次の順で復旧します:

1. `chromium\src` で `git stash push -u` してから再実行
2. `automate-git.py` に `--force-cef-update` を付けて再実行（推奨）
3. それでも解消しない場合は `D:\cef_build_env\chromium_git\chromium\src\cef` を削除して再実行

`[FATAL:extensions\\common\\url_pattern.cc:175] ... simple_feature.cc:616`
で落ちる場合は、`chrome_ui_scheme_runtime_compat.patch` を最新版に同期して再ビルドしてください。
暫定回避として `cefclient` 起動時に `--disable-extensions` を付けると起動確認を進められます。

## 5. 漏れ確認チェック（最低限）

`cefclient` は URL を **位置引数ではなく `--url=`** で渡す:

```bat
set OUT=D:\cef_build_env\chromium_git\chromium\src\out\Release_GN_x64

%OUT%\cefclient.exe --url=rwsb://settings/ --user-data-dir=%OUT%\tmp_profile_rwsb_settings --cache-path=%OUT%\tmp_profile_rwsb_settings\cache
%OUT%\cefclient.exe --url=rwsb://version/ --user-data-dir=%OUT%\tmp_profile_rwsb_version --cache-path=%OUT%\tmp_profile_rwsb_version\cache
%OUT%\cefclient.exe --url=rwsb://downloads/ --user-data-dir=%OUT%\tmp_profile_rwsb_downloads --cache-path=%OUT%\tmp_profile_rwsb_downloads\cache
%OUT%\cefclient.exe --url=chrome://version/ --user-data-dir=%OUT%\tmp_profile_legacy_version --cache-path=%OUT%\tmp_profile_legacy_version\cache
```

`--user-data-dir` と `--cache-path` をセットで指定して、プロファイルと process singleton 判定を明示的に固定します。

### 5.1 起動しない場合（ウィンドウが出ない場合）

`[WARNING] ... root_cache_path ... process singleton behavior` が出てコマンドだけ返る場合は、
既存プロセス再利用やプロファイル状態でウィンドウが見えないケースが多いです。

```bat
set OUT=D:\cef_build_env\chromium_git\chromium\src\out\Release_GN_x64

powershell -NoProfile -Command "Get-Process cefclient -ErrorAction SilentlyContinue | ForEach-Object { Stop-Process -Id $_.Id -Force }"
rmdir /s /q %OUT%\tmp_profile_rwsb

%OUT%\cefclient.exe --url=rwsb://settings/ ^
 --user-data-dir=%OUT%\tmp_profile_rwsb ^
 --cache-path=%OUT%\tmp_profile_rwsb\cache ^
 --initial-show-state=normal ^
 --enable-logging=stderr --v=1
```

確認ポイント:

1. `chrome://...` を入力してもアドレスバー最終表示が `rwsb://...` になる  
2. `settings/version/downloads` で表示崩れ・空白化がない  
3. コンソールに CSP の `chrome://... violates ...` が連発しない

## 6. 既知の設計意図（今回の肝）

ソース全体の `chrome://` 文字列を全件機械置換せず、次の2層で互換吸収しています:

1. **正規化層**: `chrome://` 入力は `rwsb://` に寄せる  
2. **実行時互換層**: 残存する `chrome://` / `chrome-untrusted://` を配信時に `rwsb://` / `rwsb-untrusted://` へ変換

このため、将来アップデート時は「パッチ再適用 + `patch_updater --resave`」で追従しやすい構成です。
