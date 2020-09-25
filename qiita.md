# はじめに
あるプロジェクトで、Rails (APIサーバー) MySQL (DBサーバー) x Nuxt (webサーバー)を使うことになったので、Docker を使って環境構築することにしました。
ログを残すと同時に、なるべく汎用的に再利用可能な形で、環境構築をしようと思います。

「再利用可能な形」というのも、以前他のプロジェクトでLaravel を使用した際に[laradock](https://laradock.io/) で環境構築をしたのですが、環境変数にアプリケーション名や、ポート番号を指定するだけで、よしなにコンテナ等をビルドしてくれます。今回はRails版のそれを作っていきます。（ちなみにlaradockには nuxt用のコンテナを立ち上げるDockerfileは用意されていません。）

今回やってることとしては、環境変数から値を読み取って、Dockerfile 及び docker-compose.ymlを設定していくだけです。

# ディレクトリ構成
```
.
|- mysql
|   |- Dockerfile
|   └ my.cnf
|- nuxt
|   |- src
|   |   └ nuxtアプリケーション
|   └ Dockerfile
|- rails
|   |- src
|   |   |- Gemfile
|   |   |- Gemfile.lock
|   |   └ railsアプリケーション
|   └ Dockerfile
|- docker-compose.yml
└ .env
```

# MySQLサーバー
まずはDBサーバーを構築していきます。

## Dockerfile
```Dockerfile:mysql/Dockerfile
ARG MYSQL_VERSION
FROM mysql:${MYSQL_VERSION}

ENV TZ ${TIMEZONE}
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone && chown -R mysql:root /var/lib/mysql/

COPY my.cnf /etc/mysql/conf.d/my.cnf

RUN chmod 0444 /etc/mysql/conf.d/my.cnf

CMD ["mysqld"]

EXPOSE 3306
```

## docker-compose.yml
```yml:docker-compose.yml
version: "3.7"

services:
  mysql:
    build:
      context: ./mysql
      args:
        - MYSQL_VERSION=${MYSQL_VERSION}
    image: ${PROJECT_NAME}-mysql-img
    container_name: "${PROJECT_NAME}_mysql"
    environment:
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - TZ=${TIMEZONE}
    volumes:
      - ${DATA_PATH_HOST}/mysql:/var/lib/mysql
    ports:
      - "${MYSQL_PORT}:3306"
    user: "1000:50"
    command: --innodb-use-native-aio=0
```

## コンテナのビルド & 起動
```sh
$ docker-compose up -d mysql
```

ちなみにenvファイルはこんな感じです。
ここは個別にお好きな設定にしてください。

```.env
###########################################################
###################### General Setup ######################
###########################################################
# PROJECT NAME
PROJECT_NAME=railsdock

# TIMEZONE
TIMEZONE=Asia/Tokyo

# Choose storage path on your machine. For all storage systems
DATA_PATH_HOST=./.storage

### MYSQL #################################################
MYSQL_VERSION=8.0.21
MYSQL_DATABASE=default
MYSQL_USER=default
MYSQL_PASSWORD=secret
MYSQL_PORT=3306
MYSQL_ROOT_PASSWORD=root
MYSQL_ENTRYPOINT_INITDB=./mysql/docker-entrypoint-initdb.d

### Rails #################################################
RAILS_PORT=23450

### NUXT #################################################
NUXT_PORT=8080

```

# APIサーバー(Rails)構築
今回はNuxt（フロント）から呼び出すAPIサーバーとして、Rails環境を構築します。

## Dockerfile

```Dockerfile:rails/Dockerfile
FROM ruby:2.7.1

RUN apt-get update -qq \
  && apt-get install -y build-essential default-libmysqlclient-dev \
  && rm -rf /var/lib/apt/lists/* \
  && gem update

WORKDIR /rails/src
COPY ./src/Gemfile /rails/src/
COPY ./src/Gemfile.lock /rails/src/

RUN bundle install

EXPOSE 23450

CMD ["bundle", "exec", "rails", "server", "-p", "23450", "-b", "0.0.0.0"]
```

CMDを指定することで、コンテナを立ち上げるだけでrailsサーバーを起動するようにしてます。

## docker-compose.yml

```yml:docker-compose.yml
version: "3.7"

services:
  mysql:
    # 省略

  rails:
    container_name: ${PROJECT_NAME}-rails
    build: rails/
    image: ${PROJECT_NAME}-rails-img
    volumes:
      - ./rails:/rails
    ports:
      - "${RAILS_PORT}:23450"
    depends_on: 
      - mysql
    stdin_open: true
    tty: true
```

depends_onでmysqlを指定することで、mysqlサーバーコンテナが立ち上がってからrailsサーバーが立ち上がるようになります。

## Railsの起動

```Gemfile:rails/src/Gemfile
source 'https://rubygems.org'
gem 'rails', '6.0.3'
```

また、Gemfile.lockを作っておいてください。

```sh
touch rails/src/Gemfile.lock
```


```sh
$ docker-compose run rails rails new . --database=mysql --api
```

database.ymlにpasswordとhostを設定

```yml: rails/src/config/database.yml
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: root
  password: root # <= .envに設定したMYSQL_ROOT_PASSWORD
  host: mysql # <= docker-compose.ymlのserviceに設定したservice名
```

```sh
$ docker-compose run rails rails db:create
$ docker-compose up -d rails
```

これでRailsアプリケーションが立ち上がっているはずです。
```localhost:23450```（port番号はご自身のenvファイルに従う）にアクセスして、起動を確認してください。

# Nuxt （フロントの環境構築）
最後はNuxtでアプリケーションサーバーを構築していきます。

## Dockerfile

```Dockerfile:nuxt/Dockerfile
FROM node:14.5.0

ENV LNAG=C.UTF-8 \
    TZ=Asia/Tokyo

WORKDIR /nuxt/src

RUN apt-get update && \
    yarn global add \
                create-nuxt-app \
                nuxt \
                @vue/cli \
                pug \
                pug-plain-loader \
                node-sass \
                sass-loader

ENV HOST 0.0.0.0

EXPOSE 8080

CMD [ "yarn", "dev" ]
```

## docker-compose.yml

```yml:docker-compose.yml
version: "3.7"

services:
  mysql:
    # 省略

  rails:
    # 省略

  nuxt:
    build:
      context: ./nuxt
      dockerfile: Dockerfile
    image: ${PROJECT_NAME}-nuxt-img
    container_name: "${PROJECT_NAME}_nuxt_app"
    environment:
      - HOST=0.0.0.0
    volumes:
      - ./nuxt:/nuxt:cached
    ports:
      - ${NUXT_PORT}:8080
    stdin_open: true
    tty: true
```

## Nuxtアプリケーション作成

```sh
$ docker-compose run nuxt npx create-nuxt-app .


create-nuxt-app v3.1.0
✨  Generating Nuxt.js project in .
? Project name: src
? Programming language: JavaScript
? Package manager: Yarn
? UI framework: None
? Nuxt.js modules: Axios
? Linting tools: (Press <space> to select, <a> to toggle all, <i> to invert selection)
? Testing framework: None
? Rendering mode: Single Page App
? Deployment target: Server (Node.js hosting)
? Development tools: jsconfig.json (Recommended for VS Code)
```
Nuxt.js modulesはaxiosを選ぶことをお勧めします。
また、今回は導入していませんが、UI frameworkには、[Vuetify](https://vuetifyjs.com/ja/)を導入しておくと、いい感じのコンポーネントを簡単に利用することができるので、フロントが苦手な方には、もってこいなフレームワークだと思います。あとからでも導入可能なので、必要な時に入れてください。

## nuxtサーバーの設定

```js:nxut/src/nuxt.config.js
server: {
  port: 8080, //デフォルト: 3000
  host: '0.0.0.0' //デフォルト: localhost
},
```
nuxtはデフォルトで3000番ポート, localhostでサーバーを起動するように設定されています。
もし、ポート番号を任意で変えているなら、nuxt.config.jsに末行に上記のserver設定を追記してください。

## nuxtサーバー起動

```sh
$ docker-compose up -d nuxt
```
最初のnuxtアプリケーションのビルドには少し時間がかかります。長くても1分ほどだと思うので、少し待って、localhost:8080にアクセスしてください。
nuxtのトップページが見れていれば、開発環境は完成です。

# ちなみに
現在僕は、Docker on Vagrant で実装しています。こうすることでコンテナのビルドやホットリロードが早くなりました。（なった気がします。）
こちらもそのうちログを残そうと思います。