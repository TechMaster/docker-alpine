# alpine-consul-go (Customization)

Một images dùng để khởi động ứng dụng [Golang](https://golang.org) with [Consul][consul], dựa trên Images `alpine-consul`

## Bao gồm


- [Alpine-consul images]()
- [s6](http://skarnet.org/software/s6/)
- [Golang](https://golang.org)
- [Consul](https://www.consul.io/)

### Versions

- `1.0.0`, `latest` [(Dockerfile)](https://github.com/TechMaster/docker-alpine/examples/user-consul-go)


-----
## Kế thừa
Images với Consul
```
└─ alpine-base
   └─ alpine-consul
      ├─ alpine-consul-ui
      └─ alpine-consul-base
         ├─ alpine-consul-apache
         ├─ alpine-consul-nodejs
         ├─ alpine-consul-go (this image)
         ├─ alpine-consul-nginx
         |  └─ alpine-consul-nginx-nodejs
         └─ alpine-consul-redis
```

Images không kèm consul

```
└─ alpine-base
   ├─ alpine-apache
   ├─ alpine-confd
   |  └─ alpine-rabbitmq
   ├─ alpine-nginx
   |  └─ alpine-nginx-nodejs
   ├─ alpine-nodejs
   └─ alpine-redis
```
-----
## Sử dụng

Container sẽ được cài đặt để tự động kết nối tới Consul cluster, mặc định sẽ kết nối đến một container với tên là `consul`

- S6 có thể giữ các hệ thống chạy ổn định, khởi động lại các thành viên khi bị rớt.

- Nếu consul chết, nó sẽ được khởi động lại nếu container vẫn đang chạy.
- Trong trường hợp container bị chết, mà consul nằm trong đó là 1 leader, cluster sẽ chỉ định 1 consul khác thay leader.

-----
## Cấu hình

Mặc định, Consul khởi động bằng file script `run` tại `root/etc/services.d/consul/run` trong Images `alpine-consul`

Cấu hình  được định nghĩa trong `root/etc/consul/conf.d/bootstrap/config.json`
```
{
    "bootstrap_expect": 3,
    "server": true,
    "data_dir": "/data/consul",
    "disable_update_check": true
}
```

Thay thế file `config.json` bằng cấu hình tùy biến. Images `alpine-consul-ui` sử dụng tùy biến cấu hình để trở thành một client với việc thay đổi đường dẫn tới file config `CONSUL_CONFIG_DIR=/etc/consul.d`
```  
{
    "data_dir": "/data/consul",
    "disable_update_check": true
}
```

-----
## Consul-ui

Consul-ui được định nghĩa là một node client (không sử dụng option `"server": true`)

- Toàn bộ các container đều đi kèm consul và có thể expose ra port `8500` để liên kết với ứng dụng bên ngoài. 

- Hiện tại sử dụng `consul-ui` để làm điều đó thay vì `consul`, bởi khi scale consul, cổng `8500` sẽ bị trùng. 

- File script `run` tại `root/etc/services.d/consul/run` trong images alpine-consul-ui cũng có thêm cờ `-ui` để xuất ra giao diện web tại cổng `8500`.

Consul-ui chịu trách nhiệm tiếp nhận các yêu cầu RPC và gửi tới 1 server. Hoạt động nền duy nhất của client là tham gia kết nối vào cluster thông qua mạng LAN, điều này tiêu tốn rất ít băng thông và tài nguyên tối thiểu.

-----
## Khởi động container cùng Go app
Để khởi động ứng dụng với tùy chọn tự động restart: 

- Tạo 1 thư mục tại `/etc/services.d/app` (tên thư mục `app/` có thể thay đổi, miễn sao nó nằm trong `/etc/services.d`)

- Tạo 1 file `run` trong `app/` và gán cho nó quyền thực thi
( Tương tự file `run` trong `consul/` để khởi động `consul`)

- bên trong file `run` là câu lệnh để khởi chạy ứng dụng mong muốn

```
/etc/
└──  services.d/
    ├── app
    │   └── run
    ├── consul
    │   └── run
    └── resolver
        └── run
```
File `/etc/services.d/app/run`
```
#!/usr/bin/env sh

# cd into app folder
cd /app

# start app
exec go-app-binary;
```

Khi container khởi động, s6 sẽ tự động khởi động ứng dụng Go ở trên.

Cần đảm bảo đủ tài nguyên để ứng dụng Go có thể chạy, nếu file `.go` import thư viện, nên build nó trước khi build Images.

-----
### Đăng ký dịch vụ với Consul

Có thể đăng ký ứng dụng này với Consul, để các container khác có thể khám phá dịch vụ và sử dụng nó theo yêu cầu.

Tạo một file `/etc/consul/conf.d/app.json` (có thể đổi tên , miễn sao là một tệp `.json` trong `/etc/consul/conf.d/`) :

```
{
    "service": {
        "name": "app",
        "tags": ["app","golang"],
        "port": 4000,
        "check": {
            "id": "app",
            "name": "Golang app on port 4000"
            "http": "http://localhost/ping",
            "interval": "10s",
            "timeout": "1s"
        }
    }
}

```

File này sẽ đăng ký dịch vụ với consul, đặt tên dịch vụ là `app`. Nó cũng định nghĩa 1 health check trả về tại `GET / HTTP/1.1 Host: localhost:4000` . Consul sẽ kiểm tra health check này để thông báo trạng thái ứng dụng.
