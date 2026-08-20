# git-brink

[English](./README.md) · **日本語** · [Français](./README.fr.md)

うっかり `git add -A` したときに何が漏れるかを、実行前に表示する。

```
$ git brink ~

git-brink  /Users/you
origin: git@github.com:you/some-public-repo.git

CRITICAL  repository root is your home directory
          everything under $HOME is inside this repository's working tree
CRITICAL  58 sensitive path(s) would be staged by 'git add -A'
          .aws/                              AWS credentials
          .ssh/                              SSH keys
          .config/gh/hosts.yml               GitHub CLI token
          .s3cfg                             S3 credentials
          .zsh_history                       shell/tool history
          projects/deploy-key.pem            key material
          ... and 38 more
WARN      159 nested git repositories inside the working tree

2 critical, 1 warning(s).
```

## 何をするか

想定している事故は単純で、`git init` や `git clone` をディレクトリ1つ上で打ってしまうこと。
`$HOME` がリポジトリになり、`.ssh/` や `.aws/` が作業ツリーに入る。
ここでまだコミットもステージもされていないので、秘密情報スキャナには何も見えない。
`git add -A` した瞬間に全部持っていかれる。

git-brink はそれを事前に教える。

## インストール

```sh
curl -o /usr/local/bin/git-brink \
  https://raw.githubusercontent.com/urabexon/git-brink/main/git-brink
chmod +x /usr/local/bin/git-brink
```

`PATH` 上の `git-*` は git のサブコマンドになるので、`git brink` で呼べる。

必要なのは bash 3.2 以降（macOS 標準の bash で可）と git。
`gh` があればリモートが公開かどうかも見るが、無くても動く。

## 使い方

```sh
git brink                 # カレントディレクトリのリポジトリ
git brink ~/projects      # 指定パスのリポジトリ
git brink --no-network    # リモートの可視性チェックを省略
```

終了コードは `0` = 検出なし、`1` = 警告のみ、`2` = 重大な指摘あり。
pre-commit フックに使える。

```sh
#!/bin/sh
git brink --no-network || exit 1
```

## 検査する内容

| 重大度 | 内容 |
|----------|-------|
| CRITICAL | リポジトリのルートが `$HOME` / `$HOME` の祖先 / `/` |
| CRITICAL | 未追跡かつ非 ignore のパスで、鍵・認証情報・トークン・環境ファイル・シェル履歴に見えるもの |
| CRITICAL | 上記があり、かつリモートが公開 |
| WARN | 未追跡ファイルがあるのに `.gitignore` が無い |
| WARN | 作業ツリー内に入れ子の git リポジトリがある |
| WARN | 走査予算を使い切ったため結果が不確定 |

`.env.example` や `*.sample`、`*.template`、鍵ペアの公開側 `*.pub` は報告しない。

## 速度

未追跡ファイルを全部列挙すると、`$HOME` がルートのリポジトリでは数分かかる。
`Library/` やキャッシュ、全 `node_modules` を歩くため。
git-brink は丸ごと未追跡のディレクトリを git 側で折り畳ませ、その中へは自前で浅く、
予算付きで降りる。上の例で約3秒。

予算を使い切った場合は「問題なし」とは言わず、不確定であることを表示する。

## スキャナとの違い

ファイルの中身と履歴は見ない。
コミット済みのものをパターン照合したいなら
[gitleaks](https://github.com/gitleaks/gitleaks) や
[git-secrets](https://github.com/awslabs/git-secrets) を使う。役割が違うので併用できる。

## ライセンス

[MIT](./LICENSE)
