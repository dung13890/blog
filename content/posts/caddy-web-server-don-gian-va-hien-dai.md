---
title: "Caddy Web Server đơn giản và hiện đại"
date: 2017-12-30T16:28:33Z
tags: ["caddy", "web server", "Golang"]
description: "Chắc hẳn các bạn không còn lạ lẫm gì với nginx. Một Proxy mạnh mẽ đã được phát triển và hình thành lên webserver. Tuy nhiên hôm nay mình sẽ giới thiệu 1 webserver khác có tên là Caddy. Nó cực kỳ đơn giản, và theo đánh giá của mình và các tài liệu tìm được thì có thể trong tương lai gần nó sẽ là đối thủ đáng gờm của nginx."
---
![caddy-web-server](https://viblo.asia/uploads/5ae0d226-9bbf-410f-ad65-897164139fe7.jpg)
Chắc hẳn các bạn không còn lạ lẫm gì với nginx. Một Proxy mạnh mẽ đã được phát triển và hình thành lên webserver. Tuy nhiên hôm nay mình sẽ giới thiệu 1 webserver khác có tên là Caddy. Nó cực kỳ đơn giản, và theo đánh giá của mình và các tài liệu tìm được thì có thể trong tương lai gần nó sẽ là đối thủ đáng gờm của nginx.

# Giới thiệu
Caddy server được viết bằng Go tài liệu đầy đủ [Caddy Server](https://caddyserver.com/) là 1 open-source và đang được cộng đồng phát triển rất mạnh mẽ. Mục tiêu của nó là configuration đơn giản và mang lại hiệu năng cao cho applications của bạn. Nó hướng tới `HTTP/2.0` và mặc định sử dụng HTTPS (bạn vẫn có thể disable). Bạn không cần phải chạy nhiều script, bạn cũng không cần phải config rườm rà phức tạp. Tất cả chỉ đơn giản là 1 vài dòng ngắn ngủi. Hiện tại nó đã có đủ các tính năng của nginx gzipping, header modification, authentication, load balancing, ...và thậm chí nó còn tự động phục vụ `markdown` như HTML

# Cài đặt và cấu hình
Chém gió như vậy là đủ rồi bây giờ mình sẽ hướng dẫn cài đặt và cấu hình nó
## Cài đặt trực tiếp
Nếu các bạn cài đặt trực tiếp lên Server thì có thể tải từ [download](https://caddyserver.com/download/linux/amd64)
Giải nén ra và chép file executable `caddy` vào `/usr/local/bin/caddy`
Hoặc cài tự động `curl https://getcaddy.com | bash`
```
$ caddy -version
Caddy 0.10.3
```
Vậy là thành công rồi đó !

## Cài đặt với docker
Đôi khi các bạn muốn dev và muốn cài đặt 1 môi trường yêu cầu sử dụng webserver nhanh. thì các bạn có thể nghĩ đến docker hoặc thậm chí khi deploy lên production các bạn vẫn có thể dùng nó.
Trên hub có rất nhiều docker image cho caddy tuy nhiên mình vẫn hay sử dụng `https://hub.docker.com/r/abiosoft/caddy/` image này mình thấy nó khá ngon và ổn định. Các bạn có thể dùng docker-compose.yml
```
version: "3"

services:
  caddy:
    image: abiosoft/caddy
    restart: always
    volumes:
     - ./caddyfile:/srv  # Map folder caddyfile
    ports:
     - "80:80"     # Public port
     - "443:443"
    links:
      - services1
      - services2
      - services3
    tty: true
```
## Cấu hình
Caddy được viết bằng Go và biên dịch thành file executable cho nên có thế nói nó `run any where` trên linux
Nó có rất nhiều module và tất nhiên nhu cầu services của bạn cần module nào thì có thể include nó.

Giả sử mình có 1 domain `example.com`. ứng dụng của mình là java, node, python, ... đang mở 1 port 3000. Nhiệm vụ lúc này của mình làm thế nào forward vào port 3000 đó.
Rất đơn giản các bạn sẽ tạo 1 file `Caddyfile`
```
example.com {
    tls off
    proxy localhost:3000
}
```
Xong! Vỏn vẹn 3 dòng bây giờ hãy chạy lệnh
```sh
// cd to Caddyfile
$ caddy -port 80
Activating privacy features... done.
http://example.com

```
Nếu bạn muốn sử dụng HTTPS để dev mà ko cần phải sử dụng SSL Certificate của bên thứ 3 thất đơn giản bạn chỉ cần bật tls
```
example.com {
    tls self_signed
    proxy localhost:3000
}
```
Khởi chạy Caddy
```sh
// cd to Caddyfile
$ caddy -port 443
Activating privacy features... done.
https://example.com
```
## Cấu hình với ứng dụng php
Với ứng dụng PHP thì sao? Cũng như nginx vậy. Caddy cũng có `fastcgi`. Tất nhiên bạn cũng sẽ phải cài đặt `phpfpm`. Cấu hình nó không có gì là phức tạp trước tiên hãy xem listen của phpfpm là
`/run/php/php7.0-fpm.sock`  hay `127.0.0.1:9000`
```sh
$ ps aux | grep php-fpm
```
```
example.com {
    tls self_signed
    root ./public
    fastcgi / /run/php/php7.0-fpm.sock php {
        index index.php

    }
    errors storage/logs/caddy.log
    rewrite {
        to {path} {path}/ /index.php?{query}
    }

}
```

## Chạy daemon
Để giữ cho Caddy chạy running hiện nay có rất nhiều cách, các bạn có thể dụng những tool như PM2, forever... Tuy nhiên mình sẽ hướng dẫn thêm 1 cách khác tận dụng chính linux-systemd của server luôn.

Hãy tạo 1 file `caddy.service` với nội dung [https://github.com/mholt/caddy/blob/master/dist/init/linux-systemd/caddy.service](https://github.com/mholt/caddy/blob/master/dist/init/linux-systemd/caddy.service) trong `/etc/systemd/system/` thay thế `ExecStart` bằng exec của bạn.
```sh
$ sudo setcap cap_net_bind_service=+ep /usr/local/bin/caddy
$ sudo cp caddy.service /etc/systemd/system/
$ sudo chown root:root /etc/systemd/system/caddy.service
$ sudo chmod 644 /etc/systemd/system/caddy.service
$ sudo systemctl daemon-reload
$ sudo systemctl start caddy.service
```
![](https://viblo.asia/uploads/4a32407b-593c-4a56-888b-3e41468c9bd1.png)
# Tổng kết
Với ý kiến đánh giá cá nhân của mình thì caddy khá mới mẻ và tiện lợi. Với những ứng dụng web chỉ 1 file Caddyfile đơn giản cũng đủ làm mọi thứ rồi. Về hiệu năng và xử lý thời điểm hiện tại có thể nó vẫn chưa ngang bằng được với nginx tuy nhiên với cộng đồng phát triển mạnh và khá nhiều người quan tâm có thể 1 tương lai gần nó sẽ được cải thiện về  performance.
