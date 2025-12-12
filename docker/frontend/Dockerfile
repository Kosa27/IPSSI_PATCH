FROM node:18-alpine as build
WORKDIR /app
COPY ../../frontend/my-app/package*.json ./
RUN npm ci
COPY ../../frontend/my-app/ .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
