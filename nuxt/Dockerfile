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