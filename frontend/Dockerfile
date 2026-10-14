# Imagen de PRODUCCIÓN: compila Vite y sirve estáticos con nginx.
# (Dockerfile.dev se queda para Docker Compose / desarrollo)
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
