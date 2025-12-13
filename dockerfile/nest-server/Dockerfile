FROM node

WORKDIR /app

COPY . .

RUN yarn install

RUN yarn build

EXPOSE 3000

ENTRYPOINT ["node", "dist/main.js"]

# docker build -t nest-server .
# docker run -d -p 3000:3000 nest-server