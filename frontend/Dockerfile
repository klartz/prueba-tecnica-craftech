FROM node:17-alpine AS build

WORKDIR /app

ENV PATH /app/node_modules/.bin:$PATH

COPY package.json ./
RUN npm install
RUN npm install react-scripts@5.0.1 -g

COPY . ./
RUN npm run build

FROM node:17-alpine

WORKDIR /app

RUN npm install -g serve

COPY --from=build /app/build ./build

EXPOSE 3000
CMD ["serve", "-s", "build", "-l", "3000"]
