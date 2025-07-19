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