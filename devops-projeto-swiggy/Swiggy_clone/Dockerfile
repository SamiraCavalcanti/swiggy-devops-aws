# Multi-stage build for optimized image size
# Stage 1: Build
FROM node:16-alpine AS builder

# Set the working directory
WORKDIR /app

# Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy the rest of the application code
COPY . .

# Build the React app
RUN npm run build

# Stage 2: Production
FROM node:16-alpine

WORKDIR /app

# Copy only necessary files from builder
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/build ./build
COPY --from=builder /app/public ./public
COPY --from=builder /app/src ./src

# Expose port 3000
EXPOSE 3000

# Start your Node.js server
CMD ["npm", "start"]
