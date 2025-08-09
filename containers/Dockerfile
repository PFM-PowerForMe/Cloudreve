FROM docker.io/library/alpine:edge AS git
ARG TAG
ENV TAG=${TAG:-main}
RUN apk update && apk add --no-cache \
	git
WORKDIR /
RUN git -c advice.detachedHead=false clone --branch $TAG --depth=1 --recurse-submodules https://github.com/pfm-powerforme/cloudreve.git source
WORKDIR /source/


FROM docker.io/library/node:lts-alpine AS frontend
ARG TAG
ENV TAG=${TAG:-v0.0.1} \
    NODE_OPTIONS="--max-old-space-size=8192"
RUN apk update && apk add --no-cache \
	gcc musl-dev g++ make py3-pip \
	git zip
RUN corepack enable
WORKDIR /
COPY --from=git /source/assets/ /frontend/
WORKDIR /frontend/
RUN mkdir -p /frontend/.yarn/cache \
    && mkdir -p /frontend/.yarn/berry/cache
RUN --mount=type=cache,target=/frontend/.yarn/cache \
    --mount=type=cache,target=/frontend/.yarn/berry/cache \
    BUILD_SEQUENCIAL=1 yarn config set cacheFolder /frontend/.yarn/cache
RUN rm -rf build
RUN yarn install --network-timeout 1000000
RUN yarn version --new-version $TAG --no-git-tag-version
RUN yarn run build
RUN mkdir -p assets && mv build/ assets/
RUN zip -r - assets/build >assets.zip


FROM docker.io/library/golang:alpine AS backend
ARG TAG
ENV TAG=${TAG:-v0.0.1} \
    GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux
RUN apk update && apk add --no-cache \
	git
WORKDIR /
COPY --from=git /source/ backend/
COPY --from=frontend /frontend/assets.zip backend/application/statics/assets.zip
WORKDIR /backend/
RUN go build -a -ldflags "-s -w -X 'github.com/cloudreve/Cloudreve/v4/application/constants.BackendVersion=$TAG' -X 'github.com/cloudreve/Cloudreve/v4/application/constants.LastCommit=$(git rev-parse --short HEAD)'" -o cloudreve


FROM docker.io/library/alpine:latest AS runtime
WORKDIR /cloudreve
RUN apk update \
    && apk add --no-cache tzdata vips-tools ffmpeg libreoffice aria2 supervisor font-noto font-noto-cjk libheif libraw-tools\
    && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && mkdir -p ./data/temp/aria2 \
    && chmod -R 766 ./data/temp/aria2
ENV CR_ENABLE_ARIA2=1 \
    CR_SETTING_DEFAULT_thumb_ffmpeg_enabled=1 \
    CR_SETTING_DEFAULT_thumb_vips_enabled=1 \
    CR_SETTING_DEFAULT_thumb_libreoffice_enabled=1 \
    CR_SETTING_DEFAULT_media_meta_ffprobe=1  \
    CR_SETTING_DEFAULT_thumb_libraw_enabled=1
COPY --from=git /source/.build/aria2.supervisor.conf aria2.supervisor.conf
COPY --from=git /source/.build/entrypoint.sh entrypoint.sh
COPY --from=backend /backend/cloudreve cloudreve
RUN chmod +x cloudreve \
    && chmod +x entrypoint.sh
EXPOSE 5212 443
VOLUME ["/cloudreve/data"]
ENTRYPOINT ["sh", "./entrypoint.sh"]
