# Use NGINX alpine as a base image
FROM nginx:alpine

# Remove the default NGINX configuration file
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom configuration file
COPY nginx.conf /etc/nginx/conf.d

# Copy proxy parameters
COPY proxy_params /etc/nginx/proxy_params

# Expose port 80
EXPOSE 80

# Start NGINX
CMD ["nginx", "-g", "daemon off;"]