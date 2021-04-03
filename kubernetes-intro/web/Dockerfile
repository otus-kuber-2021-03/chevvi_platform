FROM nginx:1.19.9
WORKDIR /app
USER 1001
COPY ./content/homework.html .
COPY ./config/nginx.conf /etc/nginx/
COPY ./config/default.conf /etc/nginx/conf.d/
EXPOSE 8000
