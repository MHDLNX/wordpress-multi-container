FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json /app

RUN HTTP_PROXY="http://192.168.10.65:8080" HTTPS_PROXY="http://192.168.10.65:8080" npm install --force

COPY . /app

#CMD ["npm","start"]

RUN npm run build

FROM nginx AS serve

COPY --from=builder /app/build /app

COPY nginx.conf /etc/nginx/conf.d/default.conf

CMD ["nginx", "-g", "daemon off;"]

