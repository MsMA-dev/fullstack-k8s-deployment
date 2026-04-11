# ----- Step 1: Build the Angular application -----
FROM node:20-alpine AS build
# 1.1 Set working directory 
WORKDIR /app 
# 1.2 Install dependencies 
COPY package*.json ./
RUN npm install 
# 1.3 Copy application source code 
COPY . .
  
# 1.4 Build Production version 
RUN npm run build -- --configuration production 

# ----- Step 2: Serve using Nginx -----
FROM nginx:stable-alpine 

# 2.1 Copy built files to Nginx (/app/dist/frontend -> root html directory)
COPY --from=build /app/dist/frontend /usr/share/nginx/html 

# 2.2 Expose HTTP port 
EXPOSE 80

# 2.3 Start Nginx 
CMD ["nginx", "-g", "daemon off;"]
