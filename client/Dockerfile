FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

# ARG accepts the value passed via --build-arg or docker-compose build.args at build time
# ENV makes it available as a process environment variable during the build steps
# Both are needed because Vite reads from process.env at build time (npm run build) —
# it statically replaces import.meta.env.VITE_* in the bundle, so the value must exist
# before the build runs. Setting it at runtime (docker run -e) is too late.
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL

RUN npm run build

EXPOSE 5173
CMD [ "npm", "run", "preview" ]