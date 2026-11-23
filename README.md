# sterm (English)

A command that connects to EC2 instances over ssh through AWS Systems Manager (SSM) Session Manager.

```sh
ssh -i ~/.ssh/id_rsa ubuntu@fqdn      # previously
sterm ubuntu@EC2-instance-name        # sterm
```

sterm looks up the instance ID from the EC2 Name tag, then runs ssh using an SSM tunnel (`AWS-StartSSHSession`) as the ProxyCommand.
The instance does not need a public IP or an open inbound port 22.

## Requirements

### Local environment

- bash (works with the bash 3.2 that ships with macOS)
- OpenSSH client
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

### IAM permissions (connecting side)

- `ec2:DescribeInstances`
- `ssm:StartSession` (for the target instance and `arn:aws:ssm:*:*:document/AWS-StartSSHSession`)

### Target instance

- SSM Agent is running and the instance is registered as a Systems Manager managed instance
  (attach `AmazonSSMManagedInstanceCore` or similar to the instance profile)
- The public key used for connecting is registered in `~/.ssh/authorized_keys`
  (SSM only provides the tunnel; ssh authentication is still done with your key as usual)

## Installation

```sh
install -m 755 sterm /usr/local/bin/sterm
```

## Usage

```
sterm [--profile PROFILE] [--region REGION] [ssh options] [user@]instance-name [command ...]
```

All ssh options can be used as-is.

```sh
sterm ubuntu@web-server
sterm -i ~/.ssh/id_rsa ubuntu@web-server
sterm -L 8080:localhost:80 ubuntu@web-server     # port forwarding
sterm ubuntu@web-server 'df -h'                  # remote command
sterm ubuntu@i-0123456789abcdef0                 # specify the instance ID directly
sterm ubuntu@'web-*'                             # wildcard (pick by number if multiple match)
```

### AWS profile / region

You can specify `--profile` / `--region` before the destination.
Setting the `AWS_PROFILE` / `AWS_REGION` environment variables works the same way.

```sh
sterm --profile prod --region ap-northeast-1 ec2-user@batch-01
AWS_PROFILE=prod sterm ec2-user@batch-01
```

### scp / rsync / sftp

By telling these tools to use sterm instead of ssh, you can also transfer files.

```sh
scp -S sterm ./file.txt ubuntu@web-server:/tmp/
sftp -S sterm ubuntu@web-server
rsync -av -e sterm ./dir/ ubuntu@web-server:/tmp/dir/
```

## Details

- The host part of the destination is searched as an EC2 Name tag, and narrowed down to one `running` instance.
  - If nothing is found or none of the matches are `running`, the candidates are shown and it exits with an error.
  - If multiple instances match and you run it from a terminal, you can pick one by number. Without a terminal (scp, scripts, etc.) it exits with an error, so specify the instance ID directly.
- If an `i-xxxxxxxx` / `mi-xxxxxxxx` style ID is given, it is used as-is without searching.
- The destination host is replaced with the instance ID when passed to ssh.
  As a result, `known_hosts` records entries by instance ID. If you configure `~/.ssh/config`, match by instance ID, e.g. `Host i-*`.
- With `-v`, the mapping between the Name tag and the instance ID is displayed.

The command actually executed looks like this:

```sh
ssh -o 'ProxyCommand=aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p' \
    ubuntu@i-0123456789abcdef0
```

---

# sterm (日本語)

AWS Systems Manager (SSM) Session Manager 経由で EC2 インスタンスに ssh 接続するコマンドです。

```sh
ssh -i ~/.ssh/id_rsa ubuntu@fqdn      # これまで
sterm ubuntu@EC2インスタンス名         # sterm
```

EC2 の Name タグからインスタンス ID を引き、SSM のトンネル (`AWS-StartSSHSession`) を ProxyCommand に使って ssh を実行します。
インスタンスにパブリック IP やインバウンドの 22 番ポート開放は不要です。

## 必要なもの

### 手元の環境

- bash (macOS 標準の bash 3.2 で動作します)
- OpenSSH クライアント
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

### IAM 権限 (接続する側)

- `ec2:DescribeInstances`
- `ssm:StartSession` (対象インスタンスと `arn:aws:ssm:*:*:document/AWS-StartSSHSession` に対して)

### 接続先インスタンス

- SSM Agent が動いていて、Systems Manager のマネージドインスタンスとして登録されていること
  (インスタンスプロファイルに `AmazonSSMManagedInstanceCore` などを付与)
- 接続に使う公開鍵が `~/.ssh/authorized_keys` に登録されていること
  (SSM はトンネルを提供するだけで、ssh の認証は通常どおり鍵で行います)

## インストール

```sh
install -m 755 sterm /usr/local/bin/sterm
```

## 使い方

```
sterm [--profile PROFILE] [--region REGION] [ssh のオプション] [user@]インスタンス名 [command ...]
```

ssh のオプションはすべてそのまま使えます。

```sh
sterm ubuntu@web-server
sterm -i ~/.ssh/id_rsa ubuntu@web-server
sterm -L 8080:localhost:80 ubuntu@web-server     # ポートフォワード
sterm ubuntu@web-server 'df -h'                  # リモートコマンド
sterm ubuntu@i-0123456789abcdef0                 # インスタンス ID を直接指定
sterm ubuntu@'web-*'                             # ワイルドカード (複数ヒット時は番号で選択)
```

### AWS プロファイル / リージョン

接続先より前に `--profile` / `--region` を指定できます。
環境変数 `AWS_PROFILE` / `AWS_REGION` を設定しても同じです。

```sh
sterm --profile prod --region ap-northeast-1 ec2-user@batch-01
AWS_PROFILE=prod sterm ec2-user@batch-01
```

### scp / rsync / sftp

ssh の代わりに sterm を使わせることで、ファイル転送にも使えます。

```sh
scp -S sterm ./file.txt ubuntu@web-server:/tmp/
sftp -S sterm ubuntu@web-server
rsync -av -e sterm ./dir/ ubuntu@web-server:/tmp/dir/
```

## 動作の詳細

- 接続先のホスト部分を EC2 の Name タグとして検索し、`running` のインスタンスを 1 つに絞ります。
  - 見つからない / `running` のものがない場合は、候補を表示してエラーになります。
  - 複数ヒットした場合、端末からの実行なら番号で選択できます。端末でない場合 (scp・スクリプトなど) はエラーになるので、インスタンス ID を直接指定してください。
- `i-xxxxxxxx` / `mi-xxxxxxxx` 形式が指定された場合は検索せずにそのまま使います。
- ssh には接続先ホストをインスタンス ID に置き換えて渡します。
  そのため `known_hosts` にはインスタンス ID で登録され、`~/.ssh/config` で設定を書く場合は `Host i-*` のようにインスタンス ID でマッチさせてください。
- `-v` を付けると、Name タグとインスタンス ID の対応を表示します。

実際に実行されるのは次のようなコマンドです。

```sh
ssh -o 'ProxyCommand=aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p' \
    ubuntu@i-0123456789abcdef0
```
