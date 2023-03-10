FROM node:lts-bullseye-slim

# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN DEBIAN_FRONTEND=noninteractive apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
    # Found this in other images, not sure whether it's needed, it does not come from Playwright deps
    procps openssl libssl-dev \
		&& rm -rf /var/lib/apt/lists/* \
    && rm -rf /src/*.deb \
    && mkdir /app

### Set working directory
WORKDIR /app/builder

# Copies whole project over to the builder dir
COPY ../../ .

# Install turbo globaly
# THen Does a turbo prune in the context of the scope of the web app using the docker flag
RUN yarn global add turbo && turbo prune --scope=web --docker

# Lets now change the working directory to the app dir
WORKDIR /app

# First install dependencies (as they change less often)
COPY ../../.gitignore .gitignore

RUN cp -r /app/builder/out/json/. .  \
    && cp /app/builder/out/yarn.lock ./yarn.lock

# Then do a final yarn install and clean the cache
# Copy the final installation artifacts out of the builder dir
# Then remove the builder dir
RUN yarn install \
    && yarn cache clean \
    && cp -R /app/builder/out/full/. . \
    && rm -rf /app/builder

# Copy out the turbo.json configuration to /app as it's needed by turbo
COPY ../../turbo.json turbo.json

# Then run a turbo build which will gen our app and also the working prisma bin
RUN yarn turbo run build --filter=web...

# Create a new group/user to lock down the user that is running the web service
RUN groupadd -r appuser && useradd -r -g appuser appuser \
    && chown -R appuser:appuser /app \
    && mkdir -p /home/appuser/.cache/yarn \
    && chown -R appuser:appuser /home/appuser

# Switch to that user
USER appuser

# Then set a basic start command which can be overriden later during dev hot reloading or whatever
CMD node ./apps/web/.next/standalone/apps/web/server.js