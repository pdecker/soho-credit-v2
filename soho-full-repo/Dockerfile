FROM node:20-slim

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY dist/ ./dist/
COPY .env.example ./.env

RUN mkdir -p data logs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
