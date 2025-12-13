FROM node

WORKDIR /app

COPY . .

RUN yarn install

RUN yarn build

EXPOSE 3000

ENTRYPOINT ["yarn", "start"]

# docker build -t next-app .
# docker run -d -p 80:3000 next-app