# リリース手順（amd64）

### 0) 前提（初回だけ）

```bash
# 必要ツール
sudo apt install -y dpkg-dev gzip

# いま居るブランチと配置を確認（gh-pages のルート直下で実行すること）
git rev-parse --abbrev-ref HEAD   # → gh-pages
ls -1 | grep -E '^(pool|dists)$'  # 両方がルート直下にあること
```

### 1) .deb を配置

`pool/main/` に新しい `.deb` を追加

### 2) Packages を再生成（ルートで実行）

```bash
rm -f dists/stable/main/binary-amd64/Packages*
dpkg-scanpackages --arch amd64 pool/main > dists/stable/main/binary-amd64/Packages
gzip -k dists/stable/main/binary-amd64/Packages
[ -f .nojekyll ] || touch .nojekyll
git add -A
git commit -m "release: update packages (amd64) v1.0.0-3"
git push
```

### 3) 公開後の検証（サーバ側）

```bash
# Packages.gz が配信されているか（200 OK）
curl -I https://hakoniwalab.github.io/apt/dists/stable/main/binary-amd64/Packages.gz

# Filename が pool/main/ で始まること（←これが核心）
curl -sL https://hakoniwalab.github.io/apt/dists/stable/main/binary-amd64/Packages.gz \
 | gzip -dc | grep '^Filename' | head

# .deb に直接アクセスできること（例）
curl -I https://hakoniwalab.github.io/apt/pool/main/hakoniwa-core_1.0.0-3_amd64.deb
```

### 4) クライアント確認（任意）

```bash
sudo apt update
apt-cache policy hakoniwa-core-full
sudo apt install -y hakoniwa-core-full
```

---

## 補足（運用のコツ）

* **実行場所**は必ず *gh-pages のルート直下*（これを外すと `Filename:` に余計なプレフィックスが入ります）。
* `.nojekyll` は初回だけでOK（毎回触っても害はありません）。
* 旧バージョン `.deb` は基本残してOK（`apt` は最新を選びます）。不要なら削除してから再生成してください。
* 将来 `arm64` を増やす場合：

  ```bash
  mkdir -p dists/stable/main/binary-arm64
  dpkg-scanpackages --arch arm64 pool/main > dists/stable/main/binary-arm64/Packages
  gzip -k dists/stable/main/binary-arm64/Packages
  git add -A && git commit -m "release: add arm64 index" && git push
  ```

## バージョンアップ手順（Maintainers only）

> **原則**：内容を変えたら **同じバージョンでの差し替えはしない**。必ず **Debianリビジョン** または **Upstreamバージョン**を上げる。

### 0) 事前準備（初回のみ）

```bash
sudo apt install -y devscripts dpkg-dev gzip
```

（推奨）`debian/rules` に Release系ビルド＋strip を指定しておく：

```make
# debian/rules
export DEB_BUILD_MAINT_OPTIONS = hardening=+all

override_dh_auto_configure:
	dh_auto_configure -- -DCMAKE_BUILD_TYPE=RelWithDebInfo
```

---

### 1) Debianリビジョンだけ上げる（ビルド差し替え・小修正）

例：`1.0.0-3` → `1.0.0-4`

```bash
# changelog をインクリメントして "unstable" で記録
DEBFULLNAME="Takashi Mori" DEBEMAIL="tmori@hakoniwa-lab.net" \
dch -i -D unstable "Rebuild: update binaries"

# リリース印（UNRELEASED を閉じる）
dch -r

# 反映確認
dpkg-parsechangelog -S Version   # => 1.0.0-4

# ビルド（バイナリのみ・署名なし）
debuild -b -us -uc
```

---

### 2) Upstreamバージョンを上げる（機能追加や互換修正）

例：`1.0.0-4` → **`1.0.1-1`**

```bash
DEBFULLNAME="Takashi Mori" DEBEMAIL="tmori@hakoniwa-lab.net" \
dch -v 1.0.1-1 -D unstable "New upstream release 1.0.1"

dch -r
debuild -b -us -uc
```

> \*\*ABI非互換（SONAME変更）\*\*がある場合は、ライブラリのパッケージ名も世代アップ
> 例：`libhakoniwa-assets1` → `libhakoniwa-assets2`（依存関係の連鎖も更新）

---

### 3) APTリポジトリ（GitHub Pages）へ公開

> ブランチ：`gh-pages`（配信ブランチ）／**ルート直下**に `pool/` と `dists/` がある前提

```bash
# 新しい .deb を配置
cp ../*.deb pool/main/

# パッケージ索引を再生成（amd64）
rm -f dists/stable/main/binary-amd64/Packages*
dpkg-scanpackages --arch amd64 pool/main > dists/stable/main/binary-amd64/Packages
gzip -kf dists/stable/main/binary-amd64/Packages

# Pages の余計な処理を無効化（初回のみでOK）
touch .nojekyll

git add -A
git commit -m "release: hakoniwa-core 1.0.0-4"
git push
```

---

### 4) 公開後の検証

```bash
# サーバ側に配信されているか
curl -I https://hakoniwalab.github.io/apt/dists/stable/main/binary-amd64/Packages.gz

# Filename が "pool/main/..." で始まること（ここが核心）
curl -sL https://hakoniwalab.github.io/apt/dists/stable/main/binary-amd64/Packages.gz \
 | gzip -dc | grep '^Filename' | head

# 個別 .deb が 200 で取れるか（例）
curl -I https://hakoniwalab.github.io/apt/pool/main/hakoniwa-core_1.0.0-4_amd64.deb
```

---

### 5) クライアント側での確認（任意）

```bash
sudo apt update
apt-cache policy hakoniwa-core-full
sudo apt upgrade
```

> どうしても「同版で入れ直し」たい検証時は（本番では非推奨）：
>
> ```bash
> sudo apt clean
> sudo rm -f /var/cache/apt/archives/hakoniwa-core_*.deb
> sudo apt install --reinstall -y hakoniwa-core
> ```

---

### 運用メモ

* **同じバージョンの `.deb` を上書き配布しない**（クライアントは更新とみなさない）
* `hakoniwa-core-full` の `Depends: (= ${binary:Version})` は **同一世代の一括アップデート**を保証
* 古い版を消す場合は、**削除後に必ず `Packages` を再生成**（残骸があると 404）
