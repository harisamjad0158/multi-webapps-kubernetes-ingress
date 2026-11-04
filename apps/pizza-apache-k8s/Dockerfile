FROM httpd:2.4-alpine

COPY apache/httpd.conf /usr/local/apache2/conf/httpd.conf

COPY --chown=10001:10001 app/ /usr/local/apache2/htdocs/

USER 10001

EXPOSE 8080
