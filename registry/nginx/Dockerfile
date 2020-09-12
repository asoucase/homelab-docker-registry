FROM nginx:1.17

# Install openssl
RUN apt update && apt install -y openssl

# Generate self-signed certificate
RUN openssl req -x509 -nodes -days 365 \
    -subj "/C=CA/ST=QC/O=myDockerRegistry, Inc./CN=example.com" \
    -addext "subjectAltName=DNS:example.com" -newkey rsa:2048 \
    -keyout /etc/ssl/private/nginx-selfsigned.key \
    -out /etc/ssl/certs/nginx-selfsigned.crt;

# Forward nginx logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log

# Copy nginx config file
COPY nginx/nginx.conf /etc/nginx/nginx.conf

# Expose port 443 for https
EXPOSE 443