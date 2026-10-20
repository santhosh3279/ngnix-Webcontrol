FROM node:22-bookworm-slim AS build
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund
COPY src ./src
COPY public ./public
COPY tsconfig.json postcss.config.mjs tailwind.config.ts .eslintrc.json ./
# Deploy at the domain root rather than the original GitHub Pages subdirectory.
RUN printf 'module.exports = { reactStrictMode: true };\n' > next.config.js
RUN find src -type f -exec sed -i -e 's|/wedding-invitation/|/|g' -e 's|wedding-invitation/images/|/images/|g' {} +
# The guestbook is optional and requires Supabase credentials, absent in this source.
RUN printf 'export default function GuestBook() { return null; }\n' > src/components/sections/GuestBook.tsx
RUN npm run build

FROM node:22-bookworm-slim
WORKDIR /app
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1
COPY --from=build --chown=node:node /app /app
USER node
EXPOSE 3000
CMD ["npm", "run", "start", "--", "--hostname", "0.0.0.0"]
