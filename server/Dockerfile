

FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

ENV PORT=5000
ENV MONGO_URI=mongodb://host.docker.internal:27017/devops-mern

COPY  . .

EXPOSE 5000

CMD ["node", "server.js"]