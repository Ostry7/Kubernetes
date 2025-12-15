FROM nginx:alpine

ENV BACKEND_URL=http://backend-service:3000

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY index.html /usr/share/nginx/html/index.html
