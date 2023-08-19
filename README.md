# Stable Diffusion XL - TorchServe

```
pip install -r requirements.txt
```

```
python3 download_model.py
bash zip_model.sh
```

```
docker pull pytorch/torchserve:0.8.1-gpu
```

```
docker run -it --rm --gpus all -v `pwd`:/opt/src pytorch/torchserve:0.8.1-gpu bash
```


```
torch-model-archiver --model-name sdxl --version 1.0 --handler sdxl_handler.py --extra-files sdxl-1.0-model.zip -r requirements.txt
```

```
docker run --rm --shm-size=1g \
        --ulimit memlock=-1 \
        --ulimit stack=67108864 \
        -p8080:8080 \
        -p8081:8081 \
        -p8082:8082 \
        -p7070:7070 \
        -p7071:7071 \
		--gpus all \
		-v /home/ubuntu/sdxl-torchserve/config.properties:/home/model-server/config.properties \
        --mount type=bind,source=/home/ubuntu/sdxl-torchserve,target=/tmp/models pytorch/torchserve:0.8.1-gpu torchserve --model-store=/tmp/models
```


For batching modify `config.properties`

```
#Sample config.properties. In production config.properties at /mnt/models/config/config.properties will be used
inference_address=http://0.0.0.0:8080
management_address=http://0.0.0.0:8081
metrics_address=http://0.0.0.0:8082
enable_envvars_config=true
install_py_dep_per_model=true
load_models=sdxl.mar
max_response_size=655350000
model_store=/tmp/models
default_response_timeout=600
enable_metrics_api=true
models={\
  "sdxl": {\
    "1.0": {\
        "defaultVersion": true,\
        "marName": "sdxl.mar",\
        "minWorkers": 1,\
        "maxWorkers": 1,\
        "batchSize": 4,\
        "maxBatchDelay": 1000,\
        "responseTimeout": 600\
    }\
  }\
}
```


Caddy installation for Ubuntu

```
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

`/etc/caddy/Caddyfile`
```
backend.theschoolof.ai {
        # Set this path to your site's directory.
        #root * /usr/share/caddy

        # Enable the static file server.
        #file_server

        # Another common task is to set up a reverse proxy:
        reverse_proxy localhost:9080

        # Or serve a PHP site through php-fpm:
        # php_fastcgi localhost:9000
}

imagine.theschoolof.ai {
        reverse_proxy localhost:3000
}
```

```
sudo service caddy restart
```


Dockerfile for frontend

```
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi


# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js collects completely anonymous telemetry data about general usage.
# Learn more here: https://nextjs.org/telemetry
# Uncomment the following line in case you want to disable telemetry during the build.
# ENV NEXT_TELEMETRY_DISABLED 1

RUN yarn build

# If using npm comment out above and use below instead
# RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
# Uncomment the following line in case you want to disable telemetry during runtime.
# ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000

CMD ["node", "server.js"]
```

```
docker build -t nextjs-docker .
docker run -d -p 3000:3000 nextjs-docker
```
