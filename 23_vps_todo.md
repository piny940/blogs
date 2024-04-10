## はじめに

半年ほど前からインフラを触るようになって、VPS のインスタンスを破壊・再構築することが多くなりました。
再構築するたびに記憶を掘り起こしながらセットアップするのも面倒なので、この記事に手順をまとめておきます。

## 手順

### 1. root ユーザーでログイン

大抵の VPS はインスタンス作成時に秘密鍵をダウンロードするため、その鍵を使ってログインします。

```bash
ssh -i /path/to/private_key root@ip_address
```

場合によっては次のようなエラーが出ます。

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0664 for '/path/to/private_key' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
```

この場合、秘密鍵のパーミッションを変更します。

```bash
chmod 600 /path/to/private_key
```

再度先程のコマンドを実行すると無事ログインできます。

### 2. パッケージのアップデート

サーバーにログインしたらまずパッケージを最新にします。

```bash
apt update
apt upgrade
```

### 3. ユーザーの追加

root ユーザーは権限が強すぎるため、通常の作業は別のユーザーで行います。

```bash
adduser username
usermod -aG sudo username
```

Full Name や Room Number などは空でも構いません。
ただし、パスワードは強力なものにしてください。(この段階ではパスワードによるログインを禁止していないためパスワードが脆弱だと危険です)

### 4. SSH の設定

ローカルのマシンから秘密鍵を用いて作業用ユーザーにログインできるように、公開鍵を VPS に登録します。

ローカルマシンに鍵がない場合は作成します。

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ed25519
```

公開鍵を確認してコピーします。

```bash
cat ~/.ssh/ed25519.pub
```

VPS にログインして作業用ユーザーのディレクトリに `.ssh` ディレクトリを作成し、公開鍵を登録します。

```bash
login username
mkdir ~/.ssh
vi ~/.ssh/authorized_keys
```

`/etc/ssh/sshd_config`で SSH の設定を変更します。編集するポイントは次の 4 点です。

- ポート番号の変更
- root ユーザーでのログインを禁止
- 公開鍵によるログインを許可
- パスワードによるログインを禁止

```/etc/ssh/sshd_config
...
Port 44444 # ポート番号
...
PermitRootLogin no # rootログインを禁止
...
PubkeyAuthentication yes # 公開鍵によるログインを許可
...
PasswordAuthentication no # パスワードによるログインを禁止
```

変更後は SSH を再起動します。

```bash
sudo systemctl restart sshd
```

ローカルマシンに戻って`~/.ssh/config`に VPS の情報を追加します。

```
Host vps
  HostName ip_address
  User username
  IdentityFile ~/.ssh/ed25519
  Port 44444
```

これでローカルマシンから`ssh vps`でログインできるようになりました。

### 5. ファイアウォールの設定

ファイアウォールを設定して不要なポートを閉じます。

```bash
sudo ufw allow 44444
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

最後に再起動して設定を反映させます。

```bash
sudo reboot
```

### 6. (オプショナル) ホスト名を設定

マシンに名前をつけている人はホスト名を設定しましょう。

```bash
sudo hostnamectl set-hostname hostname
```

再起動で設定を適用します。

```bash
sudo reboot
```

### 7. (オプショナル) Dotfiles をダウンロード

dotfiles を github などで管理している人は VPS にも反映させましょう。

私は dotfiles の管理に[yadm](https://yadm.io)を使っているので、まずは yadm をインストールします。

```bash
sudo apt install yadm
```

`.ssh/authorized_keys`だけはサーバー上にあるので、レポジトリの状態に合わせます。

```bash
yadm checkout -- ~/.ssh/authorized_keys
```

yadm のクラス名を設定します。

```bash
yadm config local.class classname
```

### 8. (オプショナル) Starship のインストール

[starship](https://starship.rs/ja-JP/guide/)でターミナルの見た目をいい感じにします。

```bash
curl -sS https://starship.rs/install.sh | sh
```

`~/.bashrc`や`~/.zshrc`に次の行を追加します。

```bash
eval "$(starship init bash)"
```

`source ~/.bashrc`や`source ~/.zshrc`で設定を反映させます。

### 9. Github の設定

Github に SSH でアクセスするための設定を行います。
私は秘密鍵を全マシンで共有しているので、秘密鍵をローカルから scp でコピーします。

```bash
scp ~/.ssh/ed25519 hostname:~/.ssh/ed25519
scp ~/.ssh/ed25519.pub hostname:~/.ssh/ed25519.pub
```

VPS のセットアップの手順は以上になります。

## 最後に

今回は VPS のセットアップ手順をまとめました。
これで VPS を破壊してもすぐに再構築できるようになりました。
もし間違いや改善点があれば教えていただけると幸いです。
