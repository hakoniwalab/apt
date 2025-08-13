# Hakoniwa APT Repository (preview / unsigned)

このリポジトリは Hakoniwa の Debian/Ubuntu パッケージを配布します。  
※ 現在は **署名なし（trusted=yes）** のプレビュー運用です。

## 対応
- アーキテクチャ: **amd64**
- 対応ディストリビューション: Ubuntu 24.04（Debian系は環境により要検証）

## 事前設定（初回のみ）
```bash
echo "deb [trusted=yes] https://hakoniwalab.github.io/apt stable main" \
 | sudo tee /etc/apt/sources.list.d/hakoniwa.list

sudo apt update
````

## インストール対象

* **箱庭コア機能 (hakoniwa-core)**

  * `hakoniwa-core-full`（メタ／全部入り）
  * `hakoniwa-core`（基盤：CLIやデフォルトデータ）
  * `libhakoniwa-conductor1` / `libhakoniwa-assets1` / `libhakoniwa-shakoc1`
  * `hakoniwa-core-dev`（開発ヘッダ）
  * `python3-hakopy`（Pythonバインディング）

## インストール

一括導入（推奨）:

```bash
sudo apt install -y hakoniwa-core-full
```

## アップデート

```bash
sudo apt update
sudo apt upgrade
```

## アンインストール

```bash
# メタパッケージだけ外す
sudo apt remove hakoniwa-core-full

# 依存も含めて全削除（設定も削除）
sudo apt purge hakoniwa-core hakoniwa-core-dev \
  libhakoniwa-conductor1 libhakoniwa-assets1 libhakoniwa-shakoc1 \
  python3-hakopy
sudo apt autoremove
```

> 注: `/var/lib/hakoniwa/mmap` にユーザ作成ファイルが残っている場合、`purge` 後もディレクトリが残ることがあります。不要なら手動で削除してください。

## 動作確認

```bash
# 候補バージョンが見えること
apt-cache policy hakoniwa-core-full

# CLI のバージョン表示
hako-cmd --version

# Python バインディングの確認（任意）
python3 -c "import hakopy; print(hakopy.__version__)"
```

## トラブルシュート

* **404 エラーになる**
  `sudo apt update` を実行してから再度インストールしてください。改善しない場合は下記をチェック:

  ```bash
  # パッケージ索引が取得できているか
  curl -I https://hakoniwalab.github.io/apt/dists/stable/main/binary-amd64/Packages.gz
  # 直接 .deb に到達できるか（例）
  curl -I https://hakoniwalab.github.io/apt/pool/main/hakoniwa-core_1.0.0-3_amd64.deb
  ```
* **候補が出ない／古い情報を掴んでいる**

  ```bash
  sudo rm -rf /var/lib/apt/lists/*
  sudo apt clean
  sudo apt update
  ```
* **署名なしの警告が気になる**
  現在はプレビュー運用のため `trusted=yes` を使用しています。将来的に**署名付き**へ移行予定です。

---

### 署名付きへの移行予定（予告）

移行後は次のような記述に置き換えます（例）:

```bash
curl -fsSL https://hakoniwalab.github.io/apt/keyring/hakoniwa-archive-keyring.gpg \
 | sudo tee /usr/share/keyrings/hakoniwa-archive-keyring.gpg >/dev/null

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/hakoniwa-archive-keyring.gpg] \
https://hakoniwalab.github.io/apt stable main" \
 | sudo tee /etc/apt/sources.list.d/hakoniwa.list

sudo apt update
```
