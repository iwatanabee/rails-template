# Docker rails postgresql で環境構築

環境　windows macどっちも可能

## 手順
基本的に、この手順通りに作業していく

https://matsuand.github.io/docs.docker.jp.onthefly/samples/rails/

### Dockerfileの作成
ruby は以下のサイトから安定しているものを選ぶ

https://www.ruby-lang.org/ja/downloads/

```
FROM ruby:3.2.2

RUN gem install rails
USER root
RUN apt-get update -qq && apt-get install -y nodejs postgresql-client
RUN mkdir /myapp
WORKDIR /myapp
ADD Gemfile /myapp/Gemfile
ADD Gemfile.lock /myapp/Gemfile.lock
RUN bundle install
ADD . /myapp
COPY . /myapp

COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

CMD ["rails", "server", "-b", "0.0.0.0"]
```
### Gemfileの生成
```
source 'https://rubygems.org'
```
### Gemfile.lockの生成
ターミナルで以下のコマンドを打つ
```
touch Gemfile.lock
```

### entrypoint.shの生成
```
#!/bin/bash
set -e

# Rails に対応したファイル server.pid が存在しているかもしれないので削除する。
rm -f /myapp/tmp/pids/server.pid

# コンテナーのプロセスを実行する。（Dockerfile 内の CMD に設定されているもの。）
exec "$@"
```

### docker-compose.yml
```
services:
  db:
    image: postgres
    volumes:
      - ./tmp/db:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: password
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    tty: true
    volumes:
      - .:/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db
```
### プロジェクトをビルドする
```
docker-compose run web bundle install 
```
```
docker-compose build
```
```
docker-compose run --no-deps web rails new . --force --database=postgresql --skip-webpack-install
```
### 再ビルド
```
docker-compose build
```
### config/database.yml の編集
```
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  password: password
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: myapp_development

test:
  <<: *default
  database: myapp_test

production:
  <<: *default
  database: myapp_production
  username: myapp
  password: <%= ENV['MYAPP_DATABASE_PASSWORD'] %>

```

### データベースの生成
```
docker-compose run web rake db:create
```

### 起動
```
docker-compose up
```


### webページにアクセス
http://localhost:3000)http://localhost:3000


## Heroku にデプロイ

https://devcenter.heroku.com/ja/articles/getting-started-with-rails6

## おまけ
### 再ビルド
```
docker-compose down && docker-compose build && docker-compose up -d
```
### 再起動
```
docker-compose restart
```

## 実行コマンドの仕方
dockerの中でrails コマンドを打つ
```
docker-compose run web bundle add devise
```
rspecなどでtestを回したいとき
```
docker-compose exec web bundle exec rspec
```
### docker run exec の違い
run 新しくコンテナを立ち上げて、実行<br>
exec 起動中のコンテナでコマンドを実行
## エラー集
## Run `bundle install` to install missing gems.
インストールしたのになぜかgemがない→再ビルドしましょう
bundle install のところでビルドが止まる場合は、DockerのClear/Purge data を押して再度ビルドしましょう
![image](https://github.com/iwatanabee/rails-template/assets/83575309/fa96bce3-023b-4ee6-96a5-176015fa7a04)


## git
## git push した後に、コミットを消したいとき
きれいにしたい場合は以下のコマンドを打つ
```
git reset --hard コミットID
```

```
git push -f origin main
```


