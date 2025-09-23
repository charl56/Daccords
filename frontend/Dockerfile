# docker build -t studi0426/ostudio426:frontend .
# docker run -p 127.0.0.1:8080:80 -d studi0426/ostudio426:frontend

#####
##### Docker multi-stage build : Node.js build image
#####
FROM node:lts-alpine AS build-stage

RUN apk add --no-cache git

# make the 'app' folder the current working directory
WORKDIR /app

# Copy project files and restore as distinct layers
COPY package.json ./

# install project dependencies
RUN npm install

# Copy everything else for build
COPY tsconfig*.json ./
COPY vite.config.ts ./
COPY env.d.ts ./
COPY index.html ./
COPY src/ ./src
COPY public/ ./public

# build app for production with minification
RUN npm run build


# production stage
FROM nginx:stable-alpine AS production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]