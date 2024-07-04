---
title: "Github ActionsでHugoを使ったブログのビルドとデプロイを自動化した話"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# Github ActionsでHugoを使ったブログのビルドとデプロイを自動化した話
（この記事は,筆者の個人ブログで掲載していた内容の移設です）

Hugoを使って作成し、ホスティングをgithub.ioを使って楽チン運用してみてた時の話です。


運用に際して2つのリポジトリをgithub上に持っています。
1. サイトビルド用資材を持つprivateリポジトリ
2. 公開用資材を持つpublicリポジトリ

この２つのリポジトリに対して次のような流れでブログ記事を作っています。

1. ローカルでhugo new
2. 記事を書いたらhugoしてビルド
3. サイトビルド用資材privateリポジトリにpublicを含めて全部push
4. 公開用資材publicリポジトリにpublic配下を全部push

publicディレクトリはsubmoduleにしているわけではなく、public配下でgit initしてリモートリポジトリを分けてる感じにしています。

この運用で更新していたのですが、だんだんと以下のことを面倒だなーって思っていたので頑張って解消してみました。

* hugoがインストールできる環境じゃないと公開用資材をpushできない
* 2つのリポジトリに同じようなファイルを別々にpushすること

読むとハッピーになれるかも知れない人

* Hugoでブログ書きつつGithubでホスティングしている人
* Hugoでブログ作ってみたい人
* Github Actionsでcheckoutやらpushを狙ったリポジトリにしたい人


## やりたいことの整理
gitが使えてマークダウンが環境ならブログをかけるようにするために、実現する内容を次のように定めました。

* サイトビルド用資材privateリポジトリにpushするだけで公開用資材がビルドされる
* ビルドされた公開用資材がリモート環境から公開用資材publicリポジトリにpushされる

## 実現するためにしたこと
### Github Actionでのワークフロー作成
Githubで資材を管理しているので、CICDを構築するならGithub Actionsを使ってみない手はないでしょう。ということでGithub Actionでワークフローを作成しました。

yamlのSyntaxとかGithub Actionsで使えるコマンドは公式リファレンスを見てください。基本構造はこんなイメージ
1. ワークフロー名(冒頭のname)
2. ワークフローの起動条件(onのあたり)
3. ワークフローの中身(jobs以降)
4. jobsの中はjob名→稼働環境指定(runs-on)→処理内容(steps以降)
5. stepsの中はrunとかusesで何するかをコマンドで指示

いろんな記事を参考にしながら作成したyamlファイルは以下です。

```yaml
name: Build and Deploy

on:
  push:
    branches: master

jobs:
  build-and-deploy:
    name: build and deploy public
    runs-on: ubuntu-latest
    steps:
      - name: Setup Hugo
        run: |
          sudo apt-get install build-essential procps curl file git
          /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
          test -d ~/.linuxbrew && eval "$(~/.linuxbrew/bin/brew shellenv)"
          test -d /home/linuxbrew/.linuxbrew && eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
          test -r ~/.bash_profile && echo "eval \"\$($(brew --prefix)/bin/brew shellenv)\"" >>~/.bash_profile
          echo "eval \"\$($(brew --prefix)/bin/brew shellenv)\"" >>~/.profile
          brew --version
          brew install hugo
      - name: Checkout
        uses: actions/checkout@v2
        with:
          submodules: recursive
      - name: Checkout for Deploy repo
        uses: actions/checkout@v2
        with:
          repository: hiroperu/hiroperuLifeLog
          path: public
          lfs: true
          ssh-key: ${{ secrets.自分で設定した秘密鍵の名前 }}
      - name: Setup git repository
        run: |
          git config --global user.email "user settingのEmails内にあるKeep my email addresses privateに記載しているアドレス"
          git config --global user.name "ここにはユーザー名を入れる"
          git remote set-url origin git@github.com:ユーザー名/public配下をpushしたいリポジトリ名
      - name: Build
        run: | 
          hugo
          git status
      - name: Deploy pages
        run: |
          cd public
          git add .
          git commit -m "Deploy $GITHUB_SHA by GitHub Actions"
          git push origin master

```

この辺忘れがちだなって点とハマった点を書きます。

#### ポイント1 brewでubuntuに最新のhugoを入れている点
Hugoの[公式ドキュメント](https://gohugo.io/getting-started/installing/#homebrew-linux)にある通りにbrewでインストールするようにした。
最初のapt-getはsudoつけないと怒られます。

もしバージョン指定をしたいのであればbrewでもいいし、wget使ってもいいと思う。

#### ポイント2 公開用資材リポジトリをremoteに追加する点
これ忘れがちかなと思います。ローカルから最初にpushするリポジトリで動いているので、remoteを変えてやらないと後のpushでコケます。

次にハマった点について

#### ハマり1 Buildした際に資材に変更がないとコミット資材がなくてこける
エラーとしては
```
On branch master
Your branch is up to date with 'origin/master'.

nothing to commit, working tree clean
Error: Process completed with exit code 1.
```
というエラーが出ます。

これハマった理由が、HugoのVersionによる挙動の変化でした。
2022.02.24現在は
```
hugo v0.92.2+extended
```
が最新バージョンですが、ひとつ前のv0.92.1からビルド時の挙動が変わっています。
前バージョンではhugoでビルドした際には全てが再度ビルドされていましたが、hugoコマンドに修正が入った様で、変更がなければビルドされない挙動に変更された様です。
ちょっとコードベースでどこが変わったから、ビルドされない挙動になったのか示せず、スキル不足を実感しています。go言語勉強しなきゃですね。

このおかげで、deploy.ymlだけを変更してpushして挙動を確かめようとしていてずっとこけていました。

#### ハマり2 秘密鍵の環境変数名typo
typoは本当に気をつけましょう。特に深夜の作業のとき。。。

最後に直したいところをちょろっと書きます。

### 修正したい点 buildした際に変更が発生していない場合はpushしない様にする
ちょろっと判定書けばいいと思いますが、面倒でいれてないです。
でも入れないと、Actionsの実行結果が真っ赤になっちゃうので入れておいた方が気持ちいいと思います。


## おわりに
なんとかしてGithub ActionsでHugoのCICD環境っぽいものを整備できました。これでgitが使えてMarkdownが描ける環境なら、ブログ更新が1回のpushでできる様になりました。

ipadからでもブラウザで利用できるIDE（Cloud9やGitpod)を使えばMarkdownエディタとか要らずにブログを更新できそうです。

参考にさせていただいたサイト,記事様は以下です。
* [Zolaで作ったサイトをGitHub ActionsでGitHub PagesにDeployする](https://zenn.dev/block/articles/ba618e3dc90f7aff97ba)
* [GitHub Actions で別のリポジトリに git push する](https://3nan3.github.io/post/2019122201_github_actions/)

### 追記
yamlのコード内にbuild-adn-deployとタイプしてた箇所があった。教えていただきありがとうございます。修正しました。