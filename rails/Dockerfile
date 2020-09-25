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