FROM nginx:1.29.1
LABEL maintainer="Narcisse Spride"
RUN apt-get update -y  && \
    apt-get upgrade -y && \
    apt-get install -y git curl
RUN rm -Rf /usr/share/nginx/html/*
COPY ./app/static-website-example/ /usr/share/nginx/html

CMD ["nginx", "-g", "daemon off;"]
