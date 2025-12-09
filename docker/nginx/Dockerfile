FROM nginx:1.25-alpine

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf
COPY default.conf /etc/nginx/conf.d/default.conf

# Ensure proper permissions
RUN chmod 644 /etc/nginx/nginx.conf /etc/nginx/conf.d/default.conf
