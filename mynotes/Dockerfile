FROM node:22

WORKDIR /app

COPY . /app/

RUN npm install

EXPOSE 3000


CMD ["npm", "start", "--", "--host", "0.0.0.0"]
