✅ 1. Nginx ইনস্টল করা আছে কিনা চেক করুন
```bash
sudo nginx -v
```

না থাকলে ইনস্টল করুন:
```bash
sudo apt update
sudo apt install nginx -y
```

✅ 2. Firewall (UFW) চেক করুন
```bash
sudo ufw allow 'Nginx Full'
sudo ufw enable
```
✅ 3. Nginx configuration ফাইল তৈরি করুন

```bash
sudo nano /etc/nginx/sites-available/example.com
```

এখানে নিচের কনফিগ যুক্ত করুন:

```bash
server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://localhost:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}

```

or
```Bash
# HTTP থেকে HTTPS রিডাইরেক্ট
server {
    listen 80;
    server_name m.jhuuri.com;

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS with reverse proxy to localhost:3002
server {
    listen 443 ssl;
    server_name m.jhuuri.com;

    ssl_certificate     /etc/ssl/certs/cloudflare.crt;
    ssl_certificate_key /etc/ssl/private/cloudflare.key;

    location / {
        proxy_pass http://localhost:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}

```

✅ 4. Sites-enabled এ symlink তৈরি করুন
```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
```

✅ 5. Syntax ঠিক আছে কিনা চেক করুন
```bash
sudo nginx -t
```
✅ 6. Nginx রিলোড দিন
```bash
sudo systemctl reload nginx
```
✅ 7. ডোমেইন পয়েন্ট করা আছে কিনা চেক করুন
example.com যদি আপনার নিজের ডোমেইন হয়, তাহলে সেটা DNS এপয়েন্ট করতে হবে আপনার VPS IP address এ (A record হিসেবে)।

✅ 8. (Optional) Let's Encrypt SSL (HTTPS) সেটআপ করতে চাইলে
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d example.com -d www.example.com
```

✅ এখন আপনার সাইট ভিজিট করুন
```bash
http://example.com
```

## যদি শুধু IP দিয়ে দেখতে চান (No domain)
server_name এ _ দিন বা বাদ দিন:
```bash
server {
    listen 80 default_server;
    server_name _;

    location / {
        proxy_pass http://localhost:3002;
        # বাকিগুলো আগের মতোই
    }
}
```


# 🧪 কাজ করার জন্য যা নিশ্চিত করবেন:
1. ✅ DNS Configuration
আপনার ডোমেইন m.jhuuri.com এর A record → আপনার VPS IP address pointing করা আছে কিনা [Cloudflare বা যেখানে DNS থাকে]।

2. ✅ SSL Cert File Paths ঠিক আছে কিনা
Check:

```Bash
ls -l /etc/ssl/certs/cloudflare.crt
ls -l /etc/ssl/private/cloudflare.key
```
না থাকলে, Let's Encrypt দিয়েও করা যায়।

3. ✅ Nginx config টেস্ট ও reload করুন
```bash
sudo nginx -t
sudo systemctl reload nginx
```
🎯 এখন আপনি ব্রাউজারে লিখুন:
```bash
https://m.jhuuri.com
```



# let's encrption added

আপনার ফাইলগুলো বিশ্লেষণ করলাম। সমস্যাটা হচ্ছে আপনার `api.minthost.com.bd` এবং `minthost.com.bd` ফাইলে এখনো কিছু **Port 443** এবং **SSL** ব্লক সক্রিয় আছে, কিন্তু সেগুলোর সার্টিফিকেট ফাইলগুলো মিসিং। একারণেই Nginx স্টার্ট হতে পারছে না এবং Certbot এরর দিচ্ছে।

এটি ঠিক করার জন্য নিচের স্টেপগুলো ফলো করুন। আমরা ফাইলগুলোকে একদম ক্লিন করে ফেলব যাতে Certbot নিজে থেকে SSL বসাতে পারে।

### স্টেপ ১: কনফিগারেশন ফাইলগুলো ক্লিন করা

নিচের কমান্ডগুলো দিয়ে বিদ্যমান ফাইলগুলো মুছে নতুন করে পেস্ট করুন (ভয় নেই, আমি আপনার সব `proxy_pass` এবং সেটিংস ঠিক রেখেই ক্লিন ভার্সন দিচ্ছি):

**১. api.minthost.com.bd ঠিক করুন:**

```bash
nano /etc/nginx/sites-available/api.minthost.com.bd

```

পুরো ফাইলটা ডিলিট করে শুধু এইটুকু দিন:

```nginx
upstream backend_api {
    server 127.0.0.1:4000;
    keepalive 32;
}

limit_req_zone $binary_remote_addr zone=api_limit:10m rate=20r/s;
limit_conn_zone $binary_remote_addr zone=api_conn:10m;

server {
    listen 80;
    server_name api.minthost.com.bd;

    location / {
        proxy_pass http://backend_api;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }
}

```

**২. minthost.com.bd ঠিক করুন:**

```bash
nano /etc/nginx/sites-available/minthost.com.bd

```

পুরো ফাইলটা ডিলিট করে শুধু এইটুকু দিন:

```nginx
upstream frontend_app {
    server 127.0.0.1:3002;
    keepalive 32;
}

server {
    listen 80;
    server_name minthost.com.bd www.minthost.com.bd;

    location / {
        proxy_pass http://frontend_app;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }
}

```

**৩. niggaboi.minthost.com.bd ঠিক করুন:**

```bash
nano /etc/nginx/sites-available/niggaboi.minthost.com.bd

```

ফাইলটা ডিলিট করে ক্লিন করে নিন:

```nginx
upstream admin_frontend {
    server 127.0.0.1:3005;
}

server {
    listen 80;
    server_name niggaboi.minthost.com.bd;

    location / {
        proxy_pass http://admin_frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

```

---

### স্টেপ ২: Nginx রিস্টার্ট দিন

এখন Nginx কনফিগারেশন চেক করুন:

```bash
sudo nginx -t

```

যদি `syntax is ok` এবং `test is successful` দেখায়, তবে রিস্টার্ট দিন:

```bash
sudo systemctl restart nginx

```

---

### স্টেপ ৩: Certbot দিয়ে SSL ইনস্টল করুন

এখন আপনার কাঙ্ক্ষিত কমান্ডটি দিন। এটি এখন কোনো এরর ছাড়াই কাজ করবে এবং ফাইলগুলোতে অটোমেটিক SSL ব্লক যোগ করে দেবে:

```bash
sudo certbot --nginx -d minthost.com.bd -d www.minthost.com.bd -d api.minthost.com.bd -d niggaboi.minthost.com.bd

```

* প্রসেস চলাকালীন রিডাইরেক্ট অপশন চাইলে **2** (Redirect all traffic to HTTPS) সিলেক্ট করুন।

**কেন এটা কাজ করবে?**
আমরা আধা-খেঁচড়া SSL এবং Redirect ব্লকগুলো সরিয়ে ফেলেছি। এখন আপনার সব ডোমেইন পোর্ট ৮০-তে আছে। Certbot এখন খুব সহজেই নিজের চ্যালেঞ্জগুলো ভেরিফাই করতে পারবে এবং ফাইলগুলো সাজিয়ে দেবে।

কাজটি শেষ হলে আমাকে জানান, সবকিছু ব্রাউজারে লোড হচ্ছে কিনা। কোনো প্রজেক্টে কি **CORS Error** দিচ্ছে? দিলে আমি API ফাইলে CORS হেডার যোগ করে দেব।
