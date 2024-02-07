FROM httpd:2.4

# Copy the custom Apache configuration into the container
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf

# Copy HTML files into the container
COPY ./public/ /usr/local/apache2/htdocs/