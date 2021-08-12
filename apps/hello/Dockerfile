FROM node:13
WORKDIR /app
COPY package.json /app
RUN npm install
COPY . /app
CMD node src/index.js
EXPOSE 3000
