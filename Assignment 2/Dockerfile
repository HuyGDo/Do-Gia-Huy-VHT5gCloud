FROM nginx:alpine

# Remove default nginx static assets
RUN rm -rf /usr/share/nginx/html/*

# Copy the static website
COPY website/ /usr/share/nginx/html/

# Copy nginx configuration
COPY nginx-config/nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80
EXPOSE 80