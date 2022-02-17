---
layout: post
title: さよならQiita、こんにちはhugo × github pages
which_category: tech
---


# 対象
- Qiitaから脱したい
- Qiitaにはエモ記事が書けないので、そういった記事を独自のブログの方に書きたい


# hugoとは
https://gohugo.io/

```
Hugo is one of the most popular open-source static site generators.
With its amazing speed and flexibility, Hugo makes building websites fun again.
```
Hugoは人気のあるオープンソースの静的サイトジェネレータの1つである。
半端ないスピードと柔軟性で、Hugoは再びウェブサイトの構築を楽しくするぜ。

と公式HPには書いてあります。

# インストール
プラットフォームによってインストール方法が異なります。
Mac

```
$ brew install hugo
```

Win
choco

```
$ choco install hugo -confirm
```

Win
scoop

```
$ scoop install hugo
```

Linux
snap

```
$ snap install hugo
```


などなど、、、。さまざまなPFに対応しています。
https://gohugo.io/getting-started/installing


# さっそくブログを作ってみる
## 雛形作成

雛形を作成するために以下のコマンドを実行します。

```
$ hugo new site site-demo && cd site-demo
```

なにやら生成されたようです

```
$ tree
.
├── archetypes
│   └── default.md
├── config.toml
├── content
├── data
├── layouts
├── static
└── themes

6 directories, 2 files
```

できました。現時点だと空のディレクトリが掘られているだけのようです。

## テーマの選定
なんと、サイトの雛形は完成しました。これからどんなデザインのページにするかを決定していきます。https://themes.gohugo.io/ に主要なテーマが列挙されています。今回は
[hugo-classic](URL "https://themes.gohugo.io/hugo-classic/")でdemoを進めたいと思います。

先ほど作成したthemesというディレクトリにテーマをgit submoduleの形として入れるとGOODです。

```
$ git init && git submodule add https://github.com/goodroot/hugo-classic.git themes/hugo-classic;
```
themesにはhugo-classic以外にも使いたいテーマをどんどん追加していっても良いでしょう。ただ、そのままだとどのテーマを使って良いかわからいので、config.tomlにhugo-classicを使いますということを宣言しておきましょう。


```config.toml
baseURL = "http://example.org/"
languageCode = "en-us"
title = "My New Hugo Site"
theme = "hugo-classic" #######################ここが追加した行
```

テーマの選定はなんと、以上です。


これで記事はまだないですが画面になにかしら表示されるようにはなります。早速画面を表示して見ます。

```
hugo server
```
localhost:1313にアクセス！！

![hugo-demo-starting.png](https://qiita-image-store.s3.amazonaws.com/0/155328/f6dc4c3c-b78e-61db-8729-db5db4af7679.png)


何か表示されました。まぁ、記事も書いてませんし、テーマのカスタムも行なっておりませんので、及第点とさせてください。


## 記事をついに書く
雛形作成　-> テーマ選定まで終わりました。ので、あとは記事を書くだけです。markdownで記事を書いていきます。

```
$ hugo new post/starting-demo.md
```


content/post直下にコマンドで指定したファイルが生成されました。

```
$ tree
.
├── archetypes
│   └── default.md
├── config.toml
├── content
│   └── post
│       └── starting-demo.md ######################################ここ
├── data
├── layouts
├── resources
│   └── _gen
│       ├── assets
│       └── images
├── static
└── themes
    └── hugo-classic
        ├── LICENSE.md
        ├── README.md
        ├── archetypes
        │   └── default.md
        ├── exampleSite
        │   ├── config.toml
        │   ├── content
        │   │   ├── _index.md
        │   │   └── post
        │   │       ├── 2012-01-23-juicy-code.md
        │   │       ├── 2012-04-23-hacker-with-horn.md
        │   │       ├── 2015-07-23-command-line-awesomeness.md
        │   │       └── 2018-08-30-markdown-guide.md
        │   └── static
        │       └── css
        │           └── theme-override.css
        ├── images
        │   ├── partywizard.gif
        │   ├── screenshot.png
        │   └── tn.png
        ├── layouts
        │   ├── 404.html
        │   ├── _default
        │   │   ├── list.html
        │   │   ├── single.html
        │   │   └── terms.html
        │   └── partials
        │       ├── foot_custom.html
        │       ├── footer.html
        │       ├── head_custom.html
        │       └── header.html
        ├── static
        │   └── css
        │       ├── fonts.css
        │       └── style.css
        └── theme.toml

24 directories, 27 files

```


初期状態は以下のような状態です。下書き状態のようです。

```markdown:starting-demo.md
---
title: "Starting Demo"
date: 生成した日時
draft: true
---

```


適当に内容を書いた後、``` draft: false ```にして

```
$ hugo
```
で、記事の内容を反映させます。

```
$ tree
.
├── archetypes
│   └── default.md
├── config.toml
├── content
│   └── post
│       └── starting-demo.md
├── data
├── layouts
├── public
│   ├── 404.html
│   ├── categories
│   │   ├── index.html
│   │   └── index.xml
│   ├── css
│   │   ├── fonts.css
│   │   └── style.css
│   ├── index.html
│   ├── index.xml
│   ├── sitemap.xml
│   └── tags
│       ├── index.html
│       └── index.xml
├── resources
│   └── _gen
│       ├── assets
│       └── images
├── static
└── themes
    └── hugo-classic
        ├── LICENSE.md
        ├── README.md
        ├── archetypes
        │   └── default.md
        ├── exampleSite
        │   ├── config.toml
        │   ├── content
        │   │   ├── _index.md
        │   │   └── post
        │   │       ├── 2012-01-23-juicy-code.md
        │   │       ├── 2012-04-23-hacker-with-horn.md
        │   │       ├── 2015-07-23-command-line-awesomeness.md
        │   │       └── 2018-08-30-markdown-guide.md
        │   └── static
        │       └── css
        │           └── theme-override.css
        ├── images
        │   ├── partywizard.gif
        │   ├── screenshot.png
        │   └── tn.png
        ├── layouts
        │   ├── 404.html
        │   ├── _default
        │   │   ├── list.html
        │   │   ├── single.html
        │   │   └── terms.html
        │   └── partials
        │       ├── foot_custom.html
        │       ├── footer.html
        │       ├── head_custom.html
        │       └── header.html
        ├── static
        │   └── css
        │       ├── fonts.css
        │       └── style.css
        └── theme.toml

28 directories, 37 files

```
新たにpublicディレクトリが生成され、その中にいろいろとファイルができました。


![after_post.png](https://qiita-image-store.s3.amazonaws.com/0/155328/e20a84ed-5598-1211-8d21-cee56dbf1743.png)


トップページにも記事が出てきました。このリンクをクリックすると記事のページに遷移することができます。


# github pagesで公開
いままでのプロセスで記事を簡単に書くことができます。あとはアクセスできるようにするだけです。VPSを借りて己でWEBサーバーを立てるのもひとつの手段ではあると思いますが、今回はもっと簡単なgithub pagesを利用して秒速で構築したいと思います。



## 公開ディレクトリを指定
今回はmasterブランチのdocsディレクトリ直下を公開ディレクトリとして指定します。その宣言をconfig.tomlに書きます。

```toml:config.toml
baseURL = "https://katamotokosuke.github.io/site-demo/" ####### 追加: {user_name}.github.io/{repository_name}/ を指定する
languageCode = "en-us"
title = "My New Hugo Site"
theme = "hugo-classic"
publishDir = "docs"  ######################### 追加
canonifyurls = true  ######################## 追加(相対URLを絶対URLに変換できるようにします。)

```
記述を変更した後```hugo```コマンドを実行します。

```
$ tree
.
├── archetypes
│   └── default.md
├── config.toml
├── content
│   └── post
│       └── starting-demo.md
├── data
├── docs
│   ├── 404.html
│   ├── categories
│   │   ├── index.html
│   │   └── index.xml
│   ├── css
│   │   ├── fonts.css
│   │   └── style.css
│   ├── index.html
│   ├── index.xml
│   ├── post
│   │   ├── index.html
│   │   ├── index.xml
│   │   └── starting-demo
│   │       └── index.html
│   ├── sitemap.xml
│   └── tags
│       ├── index.html
│       └── index.xml
├── layouts
├── public
│   ├── 404.html
│   ├── categories
│   │   ├── index.html
│   │   └── index.xml
│   ├── css
│   │   ├── fonts.css
│   │   └── style.css
│   ├── index.html
│   ├── index.xml
│   ├── post
│   │   ├── index.html
│   │   ├── index.xml
│   │   └── starting-demo
│   │       └── index.html
│   ├── sitemap.xml
│   └── tags
│       ├── index.html
│       └── index.xml
├── resources
│   └── _gen
│       ├── assets
│       └── images
├── static
└── themes
    └── hugo-classic
        ├── LICENSE.md
        ├── README.md
        ├── archetypes
        │   └── default.md
        ├── exampleSite
        │   ├── config.toml
        │   ├── content
        │   │   ├── _index.md
        │   │   └── post
        │   │       ├── 2012-01-23-juicy-code.md
        │   │       ├── 2012-04-23-hacker-with-horn.md
        │   │       ├── 2015-07-23-command-line-awesomeness.md
        │   │       └── 2018-08-30-markdown-guide.md
        │   └── static
        │       └── css
        │           └── theme-override.css
        ├── images
        │   ├── partywizard.gif
        │   ├── screenshot.png
        │   └── tn.png
        ├── layouts
        │   ├── 404.html
        │   ├── _default
        │   │   ├── list.html
        │   │   ├── single.html
        │   │   └── terms.html
        │   └── partials
        │       ├── foot_custom.html
        │       ├── footer.html
        │       ├── head_custom.html
        │       └── header.html
        ├── static
        │   └── css
        │       ├── fonts.css
        │       └── style.css
        └── theme.toml

36 directories, 53 files

```
docsディレクトリが生成されました。宣言をしなければpublicディレクトリが生成されます。なので、すでに作られているpublicディレクトリはこれから不要となりますので、削除しても良いと思います。


## github側の設定
- 以上の成果物をgithub上にpushします。
- リポジトリの```Setting``` (https://github.com/#{user_name}/#{repositoryn_name}/settings )の```GitHub Pages```という設定の項目を見つける。(おそらく設定を指定なければNoneという値が選択されていると思います。)
- ```master branch docs/ folder``` を選択
- save


少し時間をおくと　https://katamotokosuke.github.io/site-demo/ にアクセスできるようになりました。


# まとめ
hugoとgithub pagesを用いて簡単にQiitaを脱することができました。これでエモ記事が書けるようになりました。
この記事では扱いませんでしたが、テーマをカスタマイズすることも可能です。そうすることでよりオリジナリティのあるWebページが構築できます。


# 参考
今回のデモで作成したgithubのソース: https://github.com/katamotokosuke/site-demo
デモで作成したWebページ: https://katamotokosuke.github.io/site-demo/
Hugo: https://gohugo.io/
