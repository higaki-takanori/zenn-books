---
title: "docker buildコマンドでなんとなく指定しているcontextを理解する"
emoji: "📼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "dockerbuild"]
published: false
publication_name: "levtech"
---

# 背景

みなさんも「dockerのbuild contextについて説明してクレメンス」と突然聞かれれることありますよね

みなさんなら説明できると思うのですが、

僕は、もう疲れちゃって 全然わからなくてェ...

調べてみたのでまとめておきます

色々手助けしてくださったN先輩いつもありがとうございます(๑╹ω╹๑ )

# 結論

dockerのbuild contextとは、「dockerのbuild時にアクセスできるファイル群」です。

そのファイル群の実態は、「[アーカイブファイル](https://wa3.i-3-i.info/word11512.html)やテキストファイル」となっています。

これだけ聞いても、はて？？って感じだと思うので、[公式サイト](https://docs.docker.com/build/building/context
)を参考に説明追加していきます。

# 説明

## そもそもdocker buildとは

公式サイトによると、
> The docker build and docker buildx build commands build Docker images from a Dockerfile and a context.

dockerfileとcontextからDocker imageを作成するコマンドみたいです。

実際のコマンドは以下です。

```shell
  docker build [OPTIONS] PATH | URL | -
                         ^^^^^^^^^^^^^^
```

`^^^^^^^^^^^^^^`で指定されている部分がbuild contextを指定する部分です。

皆さんはよく

```
docker build .
```

の形で使用しているのではないでしょうか？？

## 動作の説明

`docker build` の挙動について説明します

```
.
├── index.ts
├── src/
├── Dockerfile
├── package.json
└── package-lock.json
```

のディレクトリで `docker build .`を行うと、

```mermaid
flowchart LR
  context("index.ts<br>src/<br>Dockerfile<br>package.json<br>package-lock.json")
    subgraph MacPC
        subgraph Linux_VM
            subgraph Docker_Process
                build_process[build_process]
            end
        end
        context
    end
    style Linux_VM fill:#99ff99,stroke:#003366
    style Docker_Process fill:#66ccff,stroke:#006600
    style build_process fill:#ffcc66,stroke:#cc6600
```

まず、
`.` で指定したbuild contextを`tar`で`tarball(アーカイブファイル)`にします

```mermaid
flowchart LR
  archive(アーカイブファイル)
    subgraph MacPC
        subgraph Linux_VM
            subgraph Docker_Process
                build_process[build_process]
            end
        end
        archive
    end
    style Linux_VM fill:#99ff99,stroke:#003366
    style Docker_Process fill:#66ccff,stroke:#006600
    style build_process fill:#ffcc66,stroke:#cc6600
```

>A plain-text file or tarball piped to the docker build command through standard input

標準入力を通して、build processにアーカイブファイルを渡します。

```mermaid
flowchart LR
  archive(アーカイブファイル)
    subgraph MacPC
        subgraph Linux_Kernel
            subgraph Docker_Process
                subgraph build_process[build_process]
                  archive
                end
            end
        end
    end
    style Linux_Kernel fill:#99ff99,stroke:#003366
    style Docker_Process fill:#66ccff,stroke:#006600
    style build_process fill:#ffcc66,stroke:#cc6600
```

そして、アーカイブファイルを展開して、Dockerfileに基づいてimageを作成していきます。

```mermaid
flowchart LR
  context("index.ts<br>src/<br>Dockerfile<br>package.json<br>package-lock.json")
    subgraph MacPC
        subgraph Linux_VM
            subgraph Docker_Process
                subgraph build_process[build_process]
                  context
                end
            end
        end
    end
    style Linux_VM fill:#99ff99,stroke:#003366
    style Docker_Process fill:#66ccff,stroke:#006600
    style build_process fill:#ffcc66,stroke:#cc6600
```


## .dockerignore

そうすると、`.dockerignore`についてもより理解が進みそうです！

公式サイトによると、

>You can use a .dockerignore file to exclude files or directories from the build context.

build contextから除去したいファイルを指定できるみたいです。

つまり、`tarball（アーカイブファイル）`にするタイミングで指定したファイルを除去する

### 具体例

```
.
├── index.ts
├── ignored.ts
├── .dockerignore
├── src/
├── Dockerfile
├── package.json
└── package-lock.json
```

```:.dockerignore
ignored.ts
```

```mermaid
flowchart LR 
context("index.ts<br>ignored.ts<br>src/<br>Dockerfile<br>package.json<br>package-lock.json")
subgraph アーカイブファイル
achived("index.ts<br>src/<br>Dockerfile<br>package.json<br>package-lock.json")
end
context --ignored.tsを除く--> アーカイブファイル
```

# まとめ

簡単にですが、docker buildについてまとめてみました

少しでも学習の助けになれば幸いです
