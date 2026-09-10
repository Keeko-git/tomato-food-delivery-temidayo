# ---- Build Stage ----
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

# ---- Runtime Stage ----
FROM node:18-alpine
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=build /app/dist ./dist
COPY --from=build /app/package*.json ./
RUN npm install --omit=dev
RUN npm install vite@5.3.4 --no-save

RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 3001
CMD ["npm", "run", "preview", "--", "--host", "0.0.0.0", "--port", "3001"]

