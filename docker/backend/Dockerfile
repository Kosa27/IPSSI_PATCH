FROM node:18-alpine
WORKDIR /app
COPY ../../backend/package*.json ./
RUN npm ci --only=production
COPY ../../backend/server.js .
EXPOSE 8000
CMD ["node", "server.js"]
