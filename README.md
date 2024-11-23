## はじめに
`Apple Silicon（M1、M2など）のMac`やラズベリーパイなどマルチプラットフォームイメージを作成するためのハウツーについて、自らの学習としてアウトプットします。

[docker-buildxとmulti-platform build周りについてまとめ](https://zenn.dev/bells17/articles/docker-buildx)

https://zenn.dev/bells17/articles/docker-buildx

上記の記事がよくまとめられているので最初に目を通しておくと良いでしょう。

この記事では[上記記事](https://zenn.dev/bells17/articles/docker-buildx)からつまみ食い的に重要な点をまとめ、また記事に書かれていないが気になった点を整理します。


## 動機
顔認識フレームワーク[`FACE01`](https://github.com/yKesamaru/FACE01_DEV)では、[Docker Image](https://hub.docker.com/u/tokaikaoninsho)を用意しています。

![](assets/2024-11-20-19-56-57.png)

ただこのイメージは`x86`用で、`Apple Silicon（M1、M2など）のMac`用ではありません。

[face01_no_gpuで"python example/simple.py"を実行すると"Illegal instruction"によりエラー発生(Mac OS) #5](https://github.com/yKesamaru/FACE01_DEV/issues/5)

FACE01のユーザー様から上記Issueを頂きました。わたしはApple SiliconのMacを持っていないため作成されたイメージのテストは出来ないのですが、用意していなければそもそもテストもして頂けないだろうと思い、マルチプラットフォームイメージの作成に取り掛かる次第です。

これによりApple Siliconだけでなく、ラズベリーパイにも対応が可能となります。

## 基礎用語の確認
- [ビルドドライバー](https://docs.docker.com/build/builders/drivers/?utm_source=chatgpt.com)

  - Dockerにおけるビルドドライバーとは「ビルドを実行するバックエンドの環境を定義し、それを選択できる機能」。デフォルトのビルドドライバーは`BuildKitライブラリ`を使用する`dockerドライバー`。
  - これに対して`docker-containerドライバー`は、`BuildKit`をコンテナとして実行する独立した環境を提供する。この環境内でビルドプロセスが実行されるため、マルチプラットフォーム対応や高度なキャッシュ管理などが可能になる。

    ![](assets/2024-11-22-16-08-17.png)

- [containerd](https://docs.docker.com/desktop/features/containerd/#what-is-containerd)
  - `containerd`は、コンテナの実行、イメージ管理、ネットワーク、およびストレージを管理するための軽量なランタイム。Dockerのバックエンドとして動作し、コンテナのライフサイクル管理を担う。
- [Image store](https://docs.docker.com/desktop/features/containerd/#image-store)
  - イメージストアはDockerのプラットフォームごとのイメージ管理を行う機能であり、ローカルのイメージストアは`docker images`コマンドで確認可能。
  - マルチプラットフォームイメージ作成には、イメージインデックス（マニフェストリスト）が必要で、`containerd`がこれを管理する。これはデフォルトのイメージストア（classic Docker image store）では実現できない。`containerd  image store`ならこれができる。（正確には出来るらしいが非効率的で普通はやらない）
  - `--load`オプションは、ビルドした単一プラットフォームのイメージをローカルのイメージストア（`docker images`で表示されるリスト）にロードするためのオプション。
  - デフォルトで`--load`を有効化するには、`docker buildx create --driver-opt default-load=true`を使用可能。ただし、マルチプラットフォームビルドには適用されない。

- ビルダーインスタンス
  - ビルドプロセスを管理・実行するための独立した環境や設定の単位。各ビルダーインスタンスは、特定のドライバー（dockerやdocker-containerなど）を使用し、ビルドプロセスの構成（設定、キャッシュ、ターゲットプラットフォームなど）を定義・管理する。
  - これにより異なるビルド環境や要件に応じて、複数のビルダーインスタンスを作成・切り替えて使用することが可能となる。
    - `docker`ドライバーを使用したビルダーインスタンスの作成
      ```bash
      docker buildx create --name mydockerbuilder --driver docker --use
      ```
    - `docker-container`ドライバーを使用するビルダーインスタンスの作成
        ```bash
        docker buildx create --name mycontainerbuilder --driver docker-container
        ```
        使用する際は、`docker buildx use mycontainerbuilder`で切り替える。

## 基本的な使い方
### Docker Buildx
Docker Buildxは、Docker CLI（コマンドラインインターフェース）のプラグインとして動作する。
#### 特徴

1. マルチプラットフォームビルド
   - 異なるプラットフォーム（例`linux/amd64`や`linux/arm64`）向けのコンテナイメージを一度にビルドし、単一のイメージインデックス（マニフェストリスト）として登録できます。
   - これにより、異なるアーキテクチャをサポートするイメージを一括で管理可能。

2. 複数のビルドドライバー対応
   - `docker`ドライバー: デフォルトのビルドドライバー。Dockerデーモン内でビルドを実行。
   - `docker-container`ドライバー: BuildKitを独立したコンテナ内で動作させることで、マルチプラットフォームビルドや高度なキャッシュ機能を利用可能。
   - `kubernetes`ドライバー: Kubernetes上でBuildKitを動作させるためのドライバー。

3. キャッシュの柔軟な活用
   - ローカルキャッシュやリモートキャッシュ（例S3バケット）を利用して、ビルド時間を短縮可能。

4. カスタマイズ可能な出力オプション
   - イメージをローカルにロード（`--load`）したり、Dockerレジストリに直接プッシュ（`--push`）したり、tarファイルとしてエクスポート（`--output type=docker,dest=...`）することが可能。

5. 分散ビルド
   - 複数のマシンやノードを使用して並列にビルドを実行可能。

#### 利用手順

1. インストールと初期設定
   Docker BuildxはDocker CLIに組み込まれているため、最新バージョンのDockerをインストールすることで利用できます。

   ```bash
   docker buildx version
   ```

2. ビルダーインスタンスの作成
   Buildxはビルダーインスタンスを使用してビルドを実行します。インスタンスを作成するには以下のコマンドを使用します

   ```bash
   docker buildx create --name mybuilder --driver docker-container --use
   ```

3. マルチプラットフォームイメージのビルド
   異なるプラットフォーム向けのイメージを一度にビルドするには、`--platform`オプションを指定します。

   ```bash
   docker buildx build --platform linux/amd64,linux/arm64 -t myimage:latest --push .
   ```

4. ローカルでのイメージ利用
   ビルドしたイメージをローカルで利用するには、`--load`オプションを指定します（単一プラットフォームのみ対応）。

   ```bash
   docker buildx build --platform linux/amd64 -t myimage:latest --load .
   ```

---

#### 制約と注意点

- マルチプラットフォームイメージは`--load`オプションではローカルにロードできません。レジストリへのプッシュ（`--push`）が必要です。
- Buildxを使用するには、Docker EngineがBuildKitを有効にしている必要があります。

---

#### 活用例

1. 単一アーキテクチャのビルドとローカル利用

   ```bash
   docker buildx build --platform linux/amd64 -t myimage:latest --load .
   ```

2. 複数アーキテクチャのビルドとリモートプッシュ

   ```bash
   docker buildx build --platform linux/amd64,linux/arm64 -t myimage:latest --push .
   ```

3. キャッシュを活用した効率的なビルド

   ```bash
   docker buildx build --cache-to type=local,dest=./cache --cache-from type=local,src=./cache .
   ```

---

#### まとめ

Docker Buildxは、従来の`docker build`では対応が難しかったマルチプラットフォームビルドや分散ビルドを容易に実現する強力なツールです。柔軟なドライバー設定やキャッシュ活用などの機能を活用することで、複雑な要件にも対応可能となります。

### Docker Buildxが対応しているプラットフォームリスト
以下に、各プラットフォームの概要と主な用途を表形式でまとめました。
Docker Buildxで使用するプラットフォーム指定は、Go言語の標準に従い、**`<OS>/<アーキテクチャ>`**の形式を採用している。
これらのプラットフォームは、CPUのアーキテクチャやビット数、エンディアン（データのバイト順序）などの違いにより区別されます。

| プラットフォーム    | 概要                                    | 主な用途                                                                 |
|----------------------|-----------------------------------------|--------------------------------------------------------------------------|
| `linux/amd64`        | 64ビットのx86アーキテクチャ             | デスクトップPC、サーバー、クラウド環境など、多くのLinuxシステムで採用 |
| `linux/arm64`        | 64ビットのARMアーキテクチャ（ARMv8以降）| スマートフォン、タブレット、Raspberry Pi 3以降、AppleのM1/M2チップなど |
| `linux/arm/v7`       | 32ビットのARMアーキテクチャ（ARMv7）    | Raspberry Pi 2や一部の組み込みデバイス                                 |
| `linux/arm/v6`       | 32ビットのARMアーキテクチャ（ARMv6）    | 初代Raspberry PiやRaspberry Pi Zeroなど、古いARMデバイス               |
| `linux/386`          | 32ビットのx86アーキテクチャ             | 古いPCや一部の組み込みシステム                                          |
| `linux/ppc64le`      | 64ビットのPowerPCアーキテクチャ（リトルエンディアン） | IBMのPower Systemsなど、特定のエンタープライズサーバー                |
| `linux/s390x`        | IBMのメインフレーム向け64ビットアーキテクチャ | 金融機関や大規模企業のメインフレームシステム                           |
| `linux/riscv64`      | 64ビットのRISC-Vアーキテクチャ           | オープンソースの命令セットアーキテクチャとして、研究開発や新興のハードウェアプラットフォームで使用 |

| プラットフォーム    | 概要                                    | 主な用途                                                                 |
|----------------------|-----------------------------------------|--------------------------------------------------------------------------|
| `linux/arm/v6`       | ARMv6アーキテクチャ                    | 初代Raspberry Pi、Raspberry Pi Zero、Raspberry Pi Zero W               |
| `linux/arm/v7`       | ARMv7アーキテクチャ                    | Raspberry Pi 2                                                          |
| `linux/arm64`        | ARMv8アーキテクチャ                    | Raspberry Pi 3、3 B+、4、400、Zero 2 W、Raspberry Pi 5                  |

**ラズベリーパイ各モデルのプラットフォーム指定**:

| モデル名                 | CPUアーキテクチャ | プラットフォーム指定 |
|--------------------------|-------------------|----------------------|
| Raspberry Pi 1           | ARMv6             | `linux/arm/v6`       |
| Raspberry Pi Zero        | ARMv6             | `linux/arm/v6`       |
| Raspberry Pi Zero W      | ARMv6             | `linux/arm/v6`       |
| Raspberry Pi 2           | ARMv7             | `linux/arm/v7`       |
| Raspberry Pi 3           | ARMv8             | `linux/arm64`        |
| Raspberry Pi 3 B+        | ARMv8             | `linux/arm64`        |
| Raspberry Pi 4           | ARMv8             | `linux/arm64`        |
| Raspberry Pi 400         | ARMv8             | `linux/arm64`        |
| Raspberry Pi Zero 2 W    | ARMv8             | `linux/arm64`        |
| Raspberry Pi 5           | ARMv8             | `linux/arm64`        |

   - **注意点**:
     - **Raspberry Pi 3以降のモデル**は、64ビットのARMv8アーキテクチャを採用していますが、32ビットOSもサポートしています。そのため、32ビットOSを使用する場合は`linux/arm/v7`、64ビットOSを使用する場合は`linux/arm64`を指定します。
     - **Raspberry Pi 1とZeroシリーズ**は、ARMv6アーキテクチャのため、`linux/arm/v6`を指定します。

これらのプラットフォーム指定を正しく行うことで、Docker Buildxを使用して各デバイスに適したイメージをビルドできます。 




```bash
terms@terms:~/ドキュメント/Make_Docker_Image_Script$ docker buildx version
github.com/docker/buildx v0.17.1 257815a
terms@terms:~/ドキュメント/Make_Docker_Image_Script$ docker buildx ls
NAME/NODE     DRIVER/ENDPOINT   STATUS    BUILDKIT   PLATFORMS
default*      docker                                 
 \_ default    \_ default       running   v0.16.0    linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/386
```







Docker Imageを作成するスクリプトをブラッシュアップします。


```bash
#!/usr/bin/env bash
set -Ceux -o pipefail
IFS=$'\n\t'

# -----------------------------------------------------------------
# サマリー:
# このスクリプトは、DockerイメージのビルドおよびDocker Hubへのプッシュを自動化します。
# face01_gpuとface01_no_gpuの2種類のイメージを作成し、それぞれをDocker Hubにプッシュします。
# -----------------------------------------------------------------

function my_command() {
    # cd: FACE01_DEV/
    cd ~/bin/FACE01_DEV

    # ////////////////////////////////////////
    # face01_gpu
    # ////////////////////////////////////////

    # docker build: CPU100%になるので他の作業との兼ね合いに注意すること
    docker build -t tokaikaoninsho/face01_gpu:3.03.04 -f docker/Dockerfile_gpu . --network host
    # login
    docker login
    # docker push
    docker push tokaikaoninsho/face01_gpu:3.03.04

    # ////////////////////////////////////////
    # face01_no_gpu
    # ////////////////////////////////////////

    # docker build: CPU100%になるので他の作業との兼ね合いに注意すること
    docker build -t tokaikaoninsho/face01_no_gpu:3.03.04 -f docker/Dockerfile_no_gpu . --network host
    # login
    docker login
    # docker push
    docker push tokaikaoninsho/face01_no_gpu:3.03.04

    return 0
}


function my_error() {
    zenity --error --text="\
    失敗しました。
    "
    exit 1
}

my_command || my_error
```
自分用に用意してあるbash用のテンプレートを使った形ですが、まずは変数の設定などひと塊にしたコードにします。

```bash
#!/usr/bin/env bash

: <<'DOCSTRING'
このスクリプトは、2種類のDockerイメージ（GPU対応版と非対応版）をビルドし、
Docker Hubにプッシュするプロセスを自動化します。

- ビルド対象:
    1. face01_gpu
    2. face01_no_gpu
- 主な操作:
    - Dockerイメージのビルド
    - Docker Hubへのログイン
    - Dockerイメージのプッシュ
- 注意:
    - ビルド中にCPU使用率が高くなるため、他の作業への影響を考慮してください。
DOCSTRING

set -euo pipefail
IFS=$'\n\t'

# 定数設定
WORKDIR=~/bin/FACE01_DEV  # 作業ディレクトリ
DOCKER_REPO=tokaikaoninsho
TAG=3.03.04

# Dockerイメージをビルドしてプッシュする関数
build_and_push_image() {
    local image_name=$1   # イメージ名
    local dockerfile=$2   # 使用するDockerfile

    echo "Building Docker image: ${image_name}:${TAG}"
    docker build -t "${DOCKER_REPO}/${image_name}:${TAG}" -f "${dockerfile}" . --network host

    echo "Pushing Docker image: ${DOCKER_REPO}/${image_name}:${TAG}"
    # ログイン
    docker login
    docker push "${DOCKER_REPO}/${image_name}:${TAG}"
}

# メイン処理
main() {
    # 作業ディレクトリに移動
    cd "${WORKDIR}"

    # GPU対応版のイメージ
    build_and_push_image "face01_gpu" "docker/Dockerfile_gpu"

    # 非GPU対応版のイメージ
    build_and_push_image "face01_no_gpu" "docker/Dockerfile_no_gpu"
}

# エラー時の処理
error_handler() {
    echo "エラーが発生しました。" >&2
    exit 1
}

# 実行
trap error_handler ERR
main
```


## 参考文献
- [docker push 手順](https://zenn.dev/katan/articles/1d5ff92fd809e7)
- [grep, awkによる抽出](https://zenn.dev/sickleaf/articles/99884a12b0489cf21d45)
- [Shell Style Guide](https://github.com/google/styleguide/blob/gh-pages/shellguide.md)
- [Googleの肩に乗ってShellコーディングしちゃおう](https://qiita.com/ma91n/items/5f72ca668f1c58176644)
- [【Mac M1環境】インテルCPU用のDockerイメージをRosetta 2技術で動かす技！](https://qiita.com/naiveprince0507/items/2d268204ec753c8c6290?utm_source=chatgpt.com)
- [docker-buildxとmulti-platform build周りについてまとめ](https://zenn.dev/bells17/articles/docker-buildx)
- [`Roseeta 2`がインストールされていれば、インテルCPU向けのDockerイメージも、Mac M1で動くことが確認できました。](https://qiita.com/naiveprince0507/items/2d268204ec753c8c6290?utm_source=chatgpt.com)
- [Apple SiliconとDockerの互換性(platform)](https://zenn.dev/skrikzts/articles/8b3d2355214194?utm_source=chatgpt.com)
- [Multi-platform builds](https://docs.docker.com/build/building/multi-platform/)