# Dockerざっくり学習会

## そもそもDockerとは

Dockerとは軽量なコンテナ型アプリケーション実行環境です。
Dockerを利用することで、OS内部に独立したアプリケーションの実行環境（コンテナ）を生成することが出来
いつでもその環境を立ち上げることが可能になります。

例えばDockerを使わない旧来型の環境構築の場合
ドキュメントを参考に必要なツールをインストールするというやり方ですが
同じ環境を何度も構築するという点で相当大変です。

しかし、Dockerを利用するとコマンドを実行することで素早く同じ環境を作ることが出来ます。
複数人開発で同じ開発環境を一瞬に揃えることが出来るため、近年大変利用されています。

[参考]
https://zenn.dev/a1008u/books/6bf96a769bedb2be53ae/viewer/what_is_docker

## 基本的なDockerコマンド

- 今回のハンズオン用ディレクトリ

```
mkdir test-docker && cd test-docker
```

- Docker公式リポジトリからnginxをpullしてrun

```
docker run --rm --name test-nginx -d -p 8080:80 nginx
```

- 確認

```
curl http://localhost:8080/
docker stop test-nginx
```

### ボリュームマウント

- マウントしたいテストファイルを作成

```
cat << EOT > index.html
パズドラ王になる男
EOT
```

- ホストマシンからDockerコンテナにファイルまたはディレクトリをマウント

```
docker run --rm -v $(pwd)/index.html:/usr/share/nginx/html/index.html --name test-nginx -d -p 8080:80 nginx
```

- 確認

```
curl http://localhost:8080/
docker stop test-nginx
```

### -it

- アタッチしてないと何も見えない

```
docker run --rm ubuntu:18.04 /bin/sh
```

- 標準入力とターミナルをアタッチ

```
docker run -it --rm ubuntu:18.04 /bin/sh
```

## Dockerfile

- 簡単なDockerfileを書く

```
cat << EOT > Dockerfile
FROM alpine

COPY ./index.html /tmp/index.html

CMD ["cat", "/tmp/index.html"]
EOT
```

- ビルド

```
docker build -t test-dockerfile .
```

- ローカルイメージからrun
	- ちなみにローカルイメージがないとDocker公式リポジトリを見に行く

```
docker run --rm --name test-dockerfile test-dockerfile
```

## Dockerfileのベストプラクティス

[参考]
https://www.slideshare.net/zembutsu/explaining-best-practices-for-writing-dockerfiles

### 気にしておきたいページ

- 6,14〜17
	- レイヤーのはなし
	- 親子関係のはなし
- 25
	- ベストプラクティス
- 27
	- コンテキストのはなし
	- dockerignore
- 30
	- マルチステージビルドのはなし
- 35,39
	- RUNコマンドの性質でaptの中身が古いままになってしまう可能性がある
		- この場合はビルド時に no-cache する必要あり
		- 39のようにしたりしても

## Dockerfileの命令

### CMDとENTRYPOINTの違い

どちらもコンテナ実行時にコンテナに命令するもの
CMDはデフォルト、ENTRYPOINTは必ず実行される
CMDは上書きが可能（併用するのがベター）

- CMDは上書きできる

```
docker run --rm test-dockerfile
docker run --rm test-dockerfile ls
```

- CMDからENTRYPOINTへ

```
cat << EOT > Dockerfile
FROM alpine

COPY ./index.html /tmp/index.html

ENTRYPOINT ["cat", "/tmp/index.html"]
EOT
```

- ビルドし直し

```
docker build -t test-dockerfile .
```

- ENTRYPOINTは上書きできない

```
docker run --rm test-dockerfile
docker run --rm test-dockerfile ls
```

### CMDとRUNの違い

CMDはコンテナ起動時に実行

RUNはビルド時（中間コンテナ）に実行

## さいごに

いろいろ説明したが、結局はやりたいことを調べながら手を動かすのが一番良い
例えばDockerfileのENVは〜〜できないっていうことがやってみて分かったりする

### 最低覚えときたいコマンド

```
docker run --rm --name {container_name}
```

Docker公式イメージをつかって手元でサンドボックスにしたり
イミュータブルなビルド環境をこのコマンドで利用することが多い

```
よき例
```

### デバッグするときに

- いい感じにリスト表示

```
docker ps --format 'table {{.Names}}\t{{.ID}}'
```

```
docker images --format 'table {{.CreatedSince}}\t{{.Repository}}\t{{.Tag}}\t{{.ID}}'
```

- デバッグしたい対象の名前を確認できたら...


```
docker run --rm -it test-dockerfile /bin/bash
```

or

```
docker exec -it test-nginx /bin/bash
```

### DDLを流すとき

```
docker exec -it mysql-container mysql -uroot -p -h hogehogeDB fugafugaTable < ./piyopiyo.ddl
```

### コンテナセキュリティ

```
よき例
```

### Dockerネットワーク

- host → docker
	- localhost
- docker → docker
	- container_name
		- ※docker-composeだとservice_name ちょっとここは要検証
- docker → host
	- host.docker.internal

