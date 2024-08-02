---
title: "docker buildコマンドでなんとなく指定しているcontextを理解する"
emoji: "📼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "dockerbuild"]
published: false
publication_name: "levtech"
---

# 背景

みなさんも「dockerのbuild contextについて説明してクレメンス」と突然聞かれることありますよね。

みなさんなら説明できると思うのですが、

僕は、もう疲れちゃって 全然わからなくてェ...

調べてみたのでまとめておきます。

# 結論

dockerのbuild contextとは、「dockerのbuild時にアクセスできるファイル群」です。

そのファイル群の実態は、「[アーカイブファイル](https://wa3.i-3-i.info/word11512.html)やテキストファイル」となっています。

これだけ聞いても、はて？？って感じだと思うので、[公式サイト](https://docs.docker.com/)を参考に説明追加していきます。

:::message
今回の記事では以下を説明対象にします。

- contextの種類
    - アーカイブファイルに絞り、テキストファイルは省きます
- 環境
    - Macを対象とします。それ以外の環境は適宜読み替えてください
:::

# 説明

build contextの解説に入る前に、前提として「dockerコマンドがどのように実行されるか」を解説します。

## 前提知識

>The Docker client talks to the Docker daemon, which does the heavy lifting of building, running, and distributing your Docker containers.

Docker には、 Docker Client と Docker Host が存在しており、Docker Client は基本的にDocker Host内の Docker Daemon とやり取りをします。
 

![Docker architecture](https://docs.docker.com/guides/images/docker-architecture.webp)
>引用：[公式サイト](https://docs.docker.com/guides/docker-overview/)


また、MaxOSではLinuxVMが起動しており、その上でDocker Daemonが待機しています。[^1]

![Docker Engine Mac](https://docs.docker.jp/v1.11/_images/mac_docker_host.png)

>引用：[Docker ドキュメント日本語化プロジェクト](https://docs.docker.jp/v1.11/engine/installation/mac.html)

:::message
環境によって異なる場合があります
:::

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

`docker build` の挙動について説明します。

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
    context(["index.ts<br>src/<br>Dockerfile<br>package.json<br>package-lock.json"])
    subgraph MacPC
        subgraph Linux_VM
            subgraph Docker_Daemon
            end
        end
        subgraph Docker_Build_Process
            context
        end
    end
    style Linux_VM fill:#99ff99,stroke:#003366
    style Docker_Daemon fill:#66ccff,stroke:#006600
    style Docker_Build_Process fill:#ffcc66,stroke:#cc6600
```
>This example specifies that the PATH is ., and so tars all the files in the local directory and sends them to the Docker daemon.
https://docs.docker.com/reference/cli/docker/image/build/#build-with-path

まず、`.` で指定したbuild contextを`tar`で`tarball(アーカイブファイル)`にします。

```mermaid
flowchart LR
  archive([アーカイブファイル])
    subgraph MacPC
        subgraph Linux_VM
            subgraph Docker_Daemon
            end
        end
        subgraph Docker_Build_Process
            archive
        end
    end
    style Linux_VM fill:#99ff99,stroke:#003366
    style Docker_Daemon fill:#66ccff,stroke:#006600
    style Docker_Build_Process fill:#ffcc66,stroke:#cc6600
```

>The Docker client and daemon communicate using a REST API, over UNIX sockets or a network interface.

次に、Docker Daemon へアーカイブファイルを送信します。

Docker Daemon へは UNIXドメインソケット や TCP通信 などを通して渡されます。

```mermaid
flowchart LR
  archive([アーカイブファイル])
    subgraph MacPC
        subgraph Linux_Kernel
            subgraph Docker_Daemon
              archive
            end
        end
    end
    style Linux_Kernel fill:#99ff99,stroke:#003366
    style Docker_Daemon fill:#66ccff,stroke:#006600
%%    style Docker_Build_Process fill:#ffcc66,stroke:#cc6600
```

その後、Docker Daemon で Dockerfile と アーカイブファイルから Docker image が作成されます。

:::message
実際の作成処理は containerd が行うらしいです
:::

## .dockerignore

そうすると、`.dockerignore`についてもより理解が進みそうです！

公式サイトによると、

>You can use a .dockerignore file to exclude files or directories from the build context.

build contextから除去したいファイルを指定できるみたいです。

つまり、`tarball（アーカイブファイル）`にするタイミングで指定したファイルを除去しています。

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

簡単にですが、docker buildのcontextについてまとめてみました。

少しでも学習の助けになれば幸いです。

# 参考

[Docker公式サイト](https://docs.docker.com/)

[アーカイブファイル](https://wa3.i-3-i.info/word11512.html)

https://matsuand.github.io/docs.docker.jp.onthefly/engine/reference/commandline/build/#build-with-path

https://tech.plaid.co.jp/improve_docker_build_efficiency

https://scrapbox.io/keroxp/docker_buildを速くするコツ


[^1]: ちなみに、WindowsOSも同じようにLinuxVMが起動しており、その上でDocker Daemonが待機しています。
