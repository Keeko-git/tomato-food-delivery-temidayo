# ---- Build Stage ----
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
RUN mkdir -p uploads

# ---- Runtime Stage ----
FROM node:18-alpine
WORKDIR /app

# Create a non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only what's needed from the build stage
COPY --from=build /app ./

# Hand ownership to the non-root user
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 4000
CMD ["npm", "start"]