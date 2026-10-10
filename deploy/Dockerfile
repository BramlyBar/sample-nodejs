FROM node:22-alpine

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install \
    && rm -rf /usr/local/lib/node_modules/npm \
    && rm -rf /usr/local/bin/npm \
    && rm -rf /usr/local/bin/npx

COPY . .

EXPOSE 8080

CMD [ "node", "app.js" ]

# Open the cmd on sample-nodejs folder:
#   docker build -t my-sample-nodejs:latest -f deploy/Dockerfile .
#   docker run -d -p 8080:8080 --name running-sample-app my-sample-nodejs:latest

# On the web browser:
# http://localhost:8080/my-app (The main endpoint)
# http://localhost:8080/about (The about page)
# http://localhost:8080/metrics (The Prometheus metrics)