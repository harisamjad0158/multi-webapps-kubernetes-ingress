FROM node:20-alpine AS build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM httpd:2.4-alpine

COPY apache/httpd.conf /usr/local/apache2/conf/httpd.conf

COPY --from=build --chown=10001:10001 \
    /app/dist/ /usr/local/apache2/htdocs/

USER 10001

EXPOSE 8080
