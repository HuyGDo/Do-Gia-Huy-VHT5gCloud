FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY website/ /usr/share/nginx/html/
COPY nginx-config/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
