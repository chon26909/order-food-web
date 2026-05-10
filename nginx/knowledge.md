# nginx.conf — คำอธิบาย

## ภาพรวม

```
Client (Browser)
      │  port 80
      ▼
   nginx
      │  proxy_pass
      ▼
 Next.js app (port 3000)
```

nginx ทำหน้าที่เป็น **reverse proxy** — รับ request จาก client แล้วส่งต่อไปให้ Next.js
ไม่ได้ serve static file โดยตรง ทุก request ยังผ่าน Node.js เพื่อให้ SSR ทำงานได้

---

## upstream block

```nginx
upstream nextjs {
    server app:3000;
    keepalive 64;
}
```

| คำสั่ง | ความหมาย |
|---|---|
| `upstream nextjs` | ตั้งชื่อกลุ่ม server ปลายทาง เพื่อใช้อ้างอิงใน `proxy_pass` |
| `server app:3000` | `app` คือชื่อ service ใน docker-compose.yml — Docker DNS resolve ให้อัตโนมัติ |
| `keepalive 64` | เปิด connection pool ค้างไว้สูงสุด 64 connections ไม่ต้อง TCP handshake ใหม่ทุก request — ลด latency |

---

## Security Headers

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

| Header | ป้องกันอะไร |
|---|---|
| `X-Frame-Options: SAMEORIGIN` | ป้องกัน Clickjacking — ไม่ให้ embed หน้าเว็บใน `<iframe>` ของ domain อื่น |
| `X-Content-Type-Options: nosniff` | ป้องกัน MIME sniffing — browser ต้องเชื่อ Content-Type ที่ server ส่งมา ไม่ให้เดาเอง |
| `Referrer-Policy` | ควบคุมข้อมูลที่ส่งใน `Referer` header เมื่อคลิกลิงก์ออกนอกเว็บ |

> headers เหล่านี้ยังช่วย SEO ทางอ้อม เพราะ Google นับ security เป็นส่วนหนึ่งของ ranking signal

---

## Gzip Compression

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css ...;
```

| คำสั่ง | ความหมาย |
|---|---|
| `gzip on` | เปิดการบีบอัด response |
| `gzip_vary on` | เพิ่ม header `Vary: Accept-Encoding` — บอก CDN/cache ว่า response ต่างกันตาม encoding |
| `gzip_proxied any` | บีบอัด response ที่มาจาก upstream (Next.js) ด้วย |
| `gzip_comp_level 6` | ระดับการบีบอัด 1-9 (6 = balance ระหว่าง speed กับ size) |
| `gzip_types` | ประเภทไฟล์ที่จะบีบ — ไม่รวม image เพราะถูกบีบมาแล้ว |

> Gzip ลดขนาด HTML/CSS/JS ได้ 60-80% → ช่วย **LCP และ FCP** ซึ่งเป็น Core Web Vitals ที่ Google ใช้ rank

---

## Static Files Cache (`/_next/static/`)

```nginx
location /_next/static/ {
    proxy_pass http://nextjs;
    proxy_cache_valid 200 365d;
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

- `/_next/static/` คือ JS/CSS chunks ที่ Next.js build ออกมา
- ไฟล์เหล่านี้มี **content hash** ในชื่อ เช่น `_app-a1b2c3d4.js` — ถ้า content เปลี่ยน ชื่อไฟล์เปลี่ยนด้วย
- จึง cache ได้ยาว 1 ปี (`max-age=31536000`) อย่างปลอดภัย
- `immutable` บอก browser ว่าไม่ต้อง revalidate เลย

---

## Public Folder Cache (`/public/`)

```nginx
location /public/ {
    proxy_pass http://nextjs;
    add_header Cache-Control "public, max-age=86400";
}
```

- ไฟล์ใน `public/` เช่น รูปภาพ, favicon, robots.txt
- cache 1 วัน (`86400` วินาที) — สั้นกว่า static เพราะไม่มี hash ในชื่อ

---

## Main Proxy (SSR)

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

    proxy_read_timeout 60s;
    proxy_connect_timeout 60s;
}
```

| Header | ทำไมต้องส่ง |
|---|---|
| `Upgrade` + `Connection: upgrade` | รองรับ WebSocket (Next.js HMR ใช้ใน dev, หรือถ้าใช้ Socket.io) |
| `Host` | Next.js รู้ว่า request มาจาก domain ไหน — ใช้ใน `getServerSideProps` และ redirect |
| `X-Real-IP` | IP จริงของ client ถ้าไม่ส่ง Next.js จะเห็นแค่ IP ของ nginx |
| `X-Forwarded-For` | Chain ของ IP ทุก proxy ที่ผ่านมา — ใช้ trace หรือ rate limiting |
| `X-Forwarded-Proto` | บอก Next.js ว่า client ใช้ HTTP หรือ HTTPS — สำคัญสำหรับ redirect และ `req.url` |
| `proxy_http_version 1.1` | ต้องใช้ HTTP/1.1 เพื่อให้ `keepalive` ทำงานได้ |

---

## สรุป Request Flow

```
GET /menu  (HTTP)
  │
  ▼ nginx รับ
  │  → เติม X-Real-IP, X-Forwarded-For, X-Forwarded-Proto
  │  → gzip response ก่อนส่งกลับ
  │
  ▼ Next.js app:3000
  │  → รัน SSR: render React เป็น HTML
  │  → ส่ง HTML กลับ nginx
  │
  ▼ Client ได้ HTML สมบูรณ์
     → Googlebot อ่านได้ทันที → SEO เต็ม ✓
```
