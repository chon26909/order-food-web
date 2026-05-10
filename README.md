# Order Food Web Deployment Guide

เอกสารนี้อธิบายขั้นตอนการ deploy โปรเจกต์ Next.js นี้แบบทีละขั้น ตั้งแต่การตั้งค่า Next.js, การสร้าง Docker image, การทำงานของ nginx, docker compose และ GitHub Actions สำหรับ deploy ไปยังเครื่อง self-hosted runner

## ภาพรวมสถาปัตยกรรม

```text
GitHub push to main
        │
        ▼
GitHub Actions
        │
        ├─ Build image
        ├─ Push image ไป Docker Hub
        └─ สั่ง self-hosted runner ให้ pull image และ docker compose up
                     │
                     ▼
                nginx (port 80)
                     │
                     ▼
              Next.js app (port 3000)
```

แนวคิดของระบบนี้คือ:

- ใช้ Next.js แบบ SSR เพื่อให้ HTML ถูก render จาก server
- ใช้ Docker เก็บสภาพแวดล้อมให้เหมือนกันทุกที่
- ใช้ nginx เป็น reverse proxy หน้าแอป
- ใช้ GitHub Actions เป็นตัว build, push และ deploy

ข้อดีสำคัญคือยังคงได้ SEO เต็ม เพราะหน้าเว็บไม่ได้เป็น static-only export แต่ยัง render จาก Next.js server อยู่

---

## 1) การตั้งค่า Next.js

ไฟล์ที่เกี่ยวข้องคือ [next.config.ts](next.config.ts)

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
};

export default nextConfig;
```

### `output: "standalone"` คืออะไร

การตั้งค่านี้บอกให้ Next.js สร้าง output แบบ standalone ตอน build ซึ่งหมายความว่า:

- Next.js จะรวบรวมไฟล์ที่จำเป็นสำหรับรันโปรดักชันมาไว้ใน `.next/standalone`
- image ที่ได้จะไม่ต้องพึ่ง `node_modules` ทั้งก้อนเหมือนการรันแบบปกติ
- เหมาะกับการเอาไปใส่ใน Docker มาก
- ยังรองรับ SSR และ API routes ได้ตามปกติ

### ทำไมต้องใช้แบบนี้

ถ้าเราใช้ Docker แบบ standalone:

- image จะเล็กลง
- deploy เร็วขึ้น
- ลดโอกาสเจอปัญหา dependency ไม่ตรงกัน
- ใช้งานกับ VPS หรือ self-hosted runner ได้สะดวก

### ผลต่อ SEO

`output: "standalone"` ไม่ได้ทำให้ SEO หายไป เพราะมันยังเป็น Next.js server อยู่ ไม่ใช่ static export

---

## 2) การสร้าง Dockerfile

ไฟล์ที่เกี่ยวข้องคือ [Dockerfile](Dockerfile)

Dockerfile นี้แบ่งเป็น 3 stage:

1. `deps` สำหรับติดตั้ง dependencies
2. `builder` สำหรับ build แอป
3. `runner` สำหรับรัน production

### Stage 1: `deps`

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app

COPY package.json package-lock.json* yarn.lock* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn install --frozen-lockfile; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm install --frozen-lockfile; \
  else npm ci; \
  fi
```

จุดประสงค์ของ stage นี้คือ:

- ติดตั้ง dependencies ก่อน
- แยก layer dependencies ออกจาก source code
- ทำให้ build cache ใช้ซ้ำได้ง่ายขึ้น

### ทำไมมีการเช็คหลาย lock file

เพราะโปรเจกต์อาจใช้ package manager ไม่เหมือนกัน:

- `yarn.lock` → ใช้ Yarn
- `pnpm-lock.yaml` → ใช้ pnpm
- ถ้าไม่มีทั้งคู่ → ใช้ `npm ci`

ในโปรเจกต์นี้ถ้าใช้ `npm` เป็นหลัก `npm ci` จะเป็น path ที่ใช้งานจริง

### Stage 2: `builder`

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build
```

ขั้นตอนนี้คือ:

- เอา `node_modules` จาก stage แรกมาใช้ต่อ
- copy source code ทั้งหมดเข้าไป
- รัน `npm run build`

ผลลัพธ์คือ Next.js จะสร้าง production output รวมถึง `.next/standalone` และ `.next/static`

### Stage 3: `runner`

```dockerfile
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

stage นี้เป็น image ที่จะถูกนำไปรันจริง

#### สิ่งที่เกิดขึ้นใน runner

- สร้าง user ที่ไม่ใช่ root เพื่อความปลอดภัย
- copy `public/`
- copy output แบบ standalone
- copy static assets ของ Next.js
- รันด้วย `node server.js`

### ทำไม image นี้เหมาะกับ production

- เล็กกว่า image ที่ copy ทั้ง repo ไปหมด
- ปลอดภัยกว่าเพราะไม่รันเป็น root
- ใช้งานง่ายใน Docker Compose
- เข้ากับการ deploy บน VPS ได้ดี

---

## 3) การทำงานของ nginx

ไฟล์ที่เกี่ยวข้องคือ [nginx/nginx.conf](nginx/nginx.conf)

nginx ทำหน้าที่เป็น reverse proxy หน้า Next.js app

```nginx
upstream nextjs {
    server app:3000;
    keepalive 64;
}
```

### upstream คืออะไร

`upstream` คือชื่อกลุ่ม server ปลายทางที่ nginx จะส่ง request ไปหา

- `server app:3000` หมายถึง service ชื่อ `app` ใน docker compose
- Docker จะ resolve ชื่อนี้ผ่าน network ภายในอัตโนมัติ
- `keepalive 64` ช่วยให้ connection reuse ได้ ลด latency

### security headers

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

ความหมายแบบง่าย:

- `X-Frame-Options: SAMEORIGIN` ป้องกันเว็บถูกฝังใน iframe จากเว็บอื่น
- `X-Content-Type-Options: nosniff` ป้องกัน browser เดา MIME type เอง
- `Referrer-Policy` คุมข้อมูล referer ที่ส่งออกไป

headers เหล่านี้ช่วยเรื่องความปลอดภัย และมีผลทางอ้อมต่อคุณภาพเว็บโดยรวม

### gzip compression

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
```

nginx จะบีบอัด response ก่อนส่งกลับไปหา browser

ผลที่ได้คือ:

- HTML/CSS/JS มีขนาดเล็กลง
- โหลดเร็วขึ้น
- ช่วย Core Web Vitals เช่น FCP และ LCP

### cache สำหรับไฟล์ static

```nginx
location /_next/static/ {
    proxy_pass http://nextjs;
    proxy_cache_valid 200 365d;
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

ส่วนนี้ใช้สำหรับไฟล์ static ที่ Next.js build ออกมา เช่น JS chunk และ CSS chunk

เหตุผลที่ cache ได้ยาวคือ:

- ไฟล์เหล่านี้มี hash ในชื่อไฟล์
- ถ้า content เปลี่ยนชื่อไฟล์จะเปลี่ยนด้วย
- browser จึง cache ได้ยาวโดยไม่เสี่ยงข้อมูลเก่า

### proxy สำหรับ `/`

```nginx
location / {
    proxy_pass http://nextjs;
    proxy_http_version 1.1;

    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        "upgrade";
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

นี่คือจุดสำคัญที่สุดของการ deploy แบบ SSR

nginx จะส่งทุก request ไปให้ Next.js server render หน้า HTML ก่อน แล้วค่อยส่ง HTML กลับไปยัง browser

#### header ที่ส่งต่อ

- `Host` ทำให้แอปรู้ domain ที่เรียกเข้ามา
- `X-Real-IP` เก็บ IP จริงของ client
- `X-Forwarded-For` เก็บ chain ของ proxy ระหว่างทาง
- `X-Forwarded-Proto` บอกว่า request มาจาก HTTP หรือ HTTPS
- `Upgrade` และ `Connection` รองรับการ upgrade protocol ถ้าจำเป็น

### nginx กับ SEO

nginx เองไม่ได้ทำ SEO โดยตรง แต่มีส่วนช่วยโดย:

- ทำให้หน้าเว็บตอบสนองเร็วขึ้น
- บีบอัด response
- เก็บ cache static ได้ดี
- ส่ง request ไปให้ Next.js render HTML จริง

ส่วนที่ทำให้ SEO ดีคือการที่ Next.js ยัง render HTML จาก server ไม่ใช่ส่งแค่ JavaScript ล้วน

---

## 4) การทำงานของ docker compose

ไฟล์ที่เกี่ยวข้องคือ [docker-compose.yml](docker-compose.yml)

```yaml
services:
  app:
    image: ${DOCKER_IMAGE:-order-food-web}:${IMAGE_TAG:-latest}
    container_name: order-food-app
    restart: unless-stopped
    expose:
      - "3000"
    environment:
      - NODE_ENV=production
    networks:
      - webnet

  nginx:
    image: nginx:1.27-alpine
    container_name: order-food-nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app
    networks:
      - webnet

networks:
  webnet:
    driver: bridge
```

### service `app`

service นี้คือ Next.js production container

- ใช้ image จาก Docker Hub
- tag มาจาก `.env` หรือค่า default
- เปิด port ภายใน container ที่ `3000`
- ไม่ expose ออกนอกเครื่องโดยตรง
- มีแค่ nginx ที่เข้าถึง service นี้ผ่าน network ภายในได้

### service `nginx`

service นี้คือ public entry point

- รับ traffic ที่ port `80`
- mount nginx config จาก host เข้ามาใน container
- ส่ง request ต่อไปยัง `app`

### ทำไมต้องมี network แยก

`webnet` ทำให้ทั้งสอง container คุยกันได้ด้วยชื่อ service:

- `nginx` เรียก `app:3000`
- ไม่ต้อง hardcode IP
- ควบคุมและดูแลง่าย

### ทำไมใช้ `depends_on`

เพื่อบอกว่า nginx ควรรอให้ app ถูกสร้างก่อน

หมายเหตุ: `depends_on` ไม่ได้การันตีว่า app พร้อมตอบ request 100% ทันที แต่ช่วยเรื่องลำดับการ start ของ container

---

## 5) การ deploy ด้วย GitHub Actions

ไฟล์ที่เกี่ยวข้องคือ [.github/workflows/deploy.yml](.github/workflows/deploy.yml)

workflow นี้ทำ 2 ขั้นตอนหลัก:

1. build และ push image ไป Docker Hub
2. ให้ self-hosted runner ดึง image ใหม่แล้ว `docker compose up`

### trigger

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

หมายความว่า workflow จะรันเมื่อ:

- push เข้า `main`
- หรือกด run เองจาก GitHub Actions UI

### Job 1: build-and-push

หน้าที่ของ job นี้คือ build image จากโค้ดล่าสุดแล้ว push ไป Docker Hub

สิ่งที่เกิดขึ้นใน job นี้:

- checkout code
- setup Buildx
- login Docker Hub
- สร้าง metadata ของ image tag
- build และ push image

#### tag ที่ใช้

workflow สร้าง tag 2 แบบ:

- `sha-xxxxxxx` สำหรับผูกกับ commit นั้นโดยตรง
- `latest` สำหรับ branch หลัก

การมี tag แบบ `sha` ช่วยให้ deploy แบบระบุเวอร์ชันได้ชัดเจน

### Job 2: deploy

job นี้รันบน `self-hosted` runner

```yaml
runs-on: self-hosted
```

ความหมายคือไม่ต้อง SSH เข้า VPS จาก GitHub Actions แล้ว แต่ให้ runner ที่ติดตั้งบนเครื่องปลายทางเป็นคนรันคำสั่ง compose เอง

#### flow ของ job deploy

1. checkout repository
2. login Docker Hub
3. สร้างไฟล์ `.env` ให้ compose ใช้
4. `docker pull` image tag ล่าสุด
5. `docker compose up -d --remove-orphans`
6. `docker image prune -f` เพื่อลบ image เก่า

### ทำไมต้องมี `.env`

workflow เขียนค่า:

- `DOCKER_IMAGE`
- `IMAGE_TAG`

ลงใน `.env` เพื่อให้ `docker-compose.yml` เลือก image เวอร์ชันที่ถูกต้อง

### ทำไมใช้ self-hosted runner

เพราะต้องการ deploy บนเครื่อง VPS หรือ server ของตัวเองโดยตรง

ข้อดีคือ:

- ไม่ต้องใช้ SSH key จาก GitHub Actions
- ควบคุม environment ได้เอง
- compose ทำงานบนเครื่องปลายทางทันที

### ข้อควรมีบนเครื่อง self-hosted

ก่อน deploy ต้องมี:

- Docker
- Docker Compose plugin
- self-hosted runner ที่ online อยู่
- สิทธิ์ให้ runner รัน Docker ได้
- access ไป Docker Hub ถ้า image เป็น private

---

## 6) ขั้นตอน deploy แบบสั้นและเรียงลำดับ

### ขั้นที่ 1: ตั้งค่า Next.js

- ตั้ง `output: "standalone"` ใน [next.config.ts](next.config.ts)
- สั่ง build เพื่อให้ได้ output ที่พร้อมเอาไปใส่ใน Docker

### ขั้นที่ 2: Build image

- Dockerfile ติดตั้ง dependencies
- build แอป
- คัดเฉพาะไฟล์ที่จำเป็นไปยัง production image

### ขั้นที่ 3: ส่ง traffic ผ่าน nginx

- nginx รับ request ที่ port 80
- proxy ไปยัง Next.js app ที่ port 3000
- cache static assets และบีบอัด response

### ขั้นที่ 4: ใช้ docker compose รัน service

- `app` รัน Next.js
- `nginx` รันเป็น public gateway
- ทั้งสอง container อยู่ network เดียวกัน

### ขั้นที่ 5: ใช้ GitHub Actions deploy อัตโนมัติ

- push ไป `main`
- workflow build image และ push ไป Docker Hub
- self-hosted runner pull image ใหม่
- compose up version ล่าสุด

---

## 7) เรื่อง SEO

ถ้าถามว่า setup นี้ SEO ดีไหม คำตอบคือ **ดี** เพราะ:

- Next.js ยัง render HTML บน server
- nginx เป็นแค่ reverse proxy ไม่ได้แปลงเว็บเป็น static-only
- Googlebot เห็น content ที่ render ออกมาแล้ว
- static asset ถูก cache ได้ดี
- response เร็วขึ้นจาก gzip และ keepalive

### กรณีที่ SEO จะไม่เหมือนแบบนี้

ถ้าเปลี่ยนไปใช้ `output: "export"` แบบ static export ล้วน จะมีข้อจำกัดมากขึ้น เช่น:

- ไม่มี SSR
- ไม่มี API routes
- logic บางอย่างฝั่ง server ใช้งานไม่ได้

ดังนั้นถ้าโปรเจกต์นี้ต้องการทั้ง SEO และความยืดหยุ่นของ Next.js การใช้ `standalone` + nginx reverse proxy คือแนวทางที่เหมาะกว่า

---

## 8) สิ่งที่ต้องเตรียมก่อนใช้งานจริง

- สร้าง Docker Hub repository ให้ตรงกับชื่อ image
- ตั้ง secrets ใน GitHub Actions
- ติดตั้ง self-hosted runner บนเครื่อง deploy
- ติดตั้ง Docker และ Docker Compose บนเครื่องนั้น
- เปิด port 80 ถ้าต้องการให้เว็บเข้าจากภายนอก
- ถ้าต้องการ HTTPS ให้เพิ่ม nginx certificate ภายหลัง

---

## 9) สรุปสั้น

ระบบนี้ทำงานตามลำดับนี้:

1. Next.js build แบบ standalone
2. Docker สร้าง image production
3. nginx รับ request และ proxy ไป Next.js
4. docker compose รันสอง container
5. GitHub Actions build, push และ deploy อัตโนมัติ

ผลลัพธ์คือ deploy ได้เป็นระบบ, รันง่ายบน VPS, และยังคงได้ SEO เต็มจาก SSR
