FROM keymetrics/pm2:latest-alpine
COPY . /koa
WORKDIR /koa
ENV NPM_CONFIG_LOGLEVEL warn
RUN npm install --production  --registry https://registry.npm.taobao.org 
CMD [ "pm2-runtime", "start", "pm2start.yml" ]
