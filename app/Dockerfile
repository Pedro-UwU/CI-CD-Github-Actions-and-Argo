FROM node:17-alpine

RUN mkdir -p /home/app

COPY . /home/app

WORKDIR /home/app
RUN npm install
RUN export NODE_OPTIONS=--openssl-legacy-provider

CMD ["npm", "start"]
