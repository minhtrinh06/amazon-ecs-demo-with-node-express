FROM node:16.20.2
RUN mkdir -p /usr/src/calc
WORKDIR /usr/src/calc
COPY . .
RUN npm install
# EXPOSE 3000
CMD [ "node", "app.js" ]
