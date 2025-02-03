FROM httpd:2.4

COPY ./public-html/ /usr/local/apache2/htdocs/
COPY ./proxy.conf ./proxy.conf  
RUN echo "Include ./proxy.conf" >> /usr/local/apache2/conf/httpd.conf
