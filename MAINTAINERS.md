いいですね！その手順に、**前提チェック**と**公開後の検証**を少し足すだけで、毎回ノートラブルになります。差分で提案します。

---

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

必要なら、この手順を `MAINTAINERS.md` にそのまま貼る版も作ります！
