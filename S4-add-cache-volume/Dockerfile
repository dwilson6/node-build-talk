FROM node:21.6-slim
WORKDIR /app
COPY package.json /app/
COPY package-lock.json /app/
RUN --mount=type=cache,target=/root/.npm npm cache ls && npm install
COPY . /app
CMD node index.js