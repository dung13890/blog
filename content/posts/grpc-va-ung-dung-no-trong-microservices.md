---
title: "Grpc và ứng dụng nó trong Microservices"
date: 2017-12-27T05:40:41Z
tags: ["microservice", "GRPC", "Golang"]
description: "Hiện tại với API thì quá phổ biến cho các ứng dụng từ giao tiếp client
tới server hay từ instance tới instance."
---
Hiện tại với API thì quá phổ biến cho các ứng dụng từ giao tiếp client tới server hay từ instance tới instance. Tuy nhiên ngày nay công nghệ càng ngày càng phát triển với http2 ra đời đã kéo theo 1 loạt những thay đổi để cải thiện performance, gRPC là sự kết hợp của Protocol Buffers và http2, Protocol Buffers được phát triển bởi google nó nhẹ hơn, nhanh hơn và cung cấp hiệu năng tốt hơn so với sử dụng XML hoặc Json
gRPC cũng cho phép định nghĩa cấu trúc của data dưới dạng file protoc và nó tự động generate ra file sử dụng để giao tiếp với ngôn ngữ mà bạn sử dụng. gRPC hiện tại cũng đã hỗ trợ khá đầy đủ các ngôn ngữ như C++, Java, Python, Go, ... các bạn có thể tham khảo thêm ở đây [https://grpc.io/docs/](https://grpc.io/docs/)

![](https://viblo.asia/uploads/67f28126-e935-43c0-8e67-76d1bc1af721.jpg)

## Bài toán
Giả sử chúng ta có hệ thống microservices, Mỗi services của mình thường rất nhỏ nhưng lại cần thực thi liên tục. Services của mình quản lý thông tin người dùng, nhiệm vụ của services này chỉ thao tác các công việc liên quan đến người dùng như thêm, sửa, xoá, phân quyền, ....

Cũng 1 instance khác mình quản lý products. Thay vì theo cách thông thường, để instance products giao tiếp đến instance users lấy thông tin người dùng bằng cách sử dụng REST API chúng ta có thể thay thế nó bằng gRPC với tốc độ xử lý cực nhanh giống như việc gọi các mothod trong kiến trúc monolithic `Kiến trúc khối`

Để cụ thể chi tiết hơn mình sẽ làm 1 hướng dẫn tạo 1 file protoc với language `Go`

![](https://viblo.asia/uploads/7b48f60b-fbeb-4d69-831a-886225662ab4.jpg)

## Demo
Init một instance với docker
```sh
cd $GOPATH/src/github.com/user/examples/grpc
```

Sau đó install implementation của go
```sh
$ go get google.golang.org/grpc
```
Tiếp  tục install plugin cho Go
```sh
$ go get -u github.com/golang/protobuf/protoc-gen-go
```

Trong project mình khởi tạo 1 file protoc với đường dẫn `grpc/pb/user/user.proto`

```go
syntax = "proto3";

package user;
// Khởi tạo đối tượng User

service User {
    rpc GetUsers(GetRequest) returns (GetResponse);
    // Sử dụng giao thức rpc để gọi trực tiếp đến method của instance

    rpc FindUser(FindRequest) returns (FindResponse);

    rpc CreateUser(CreateRequest) returns (CreateResponse);
}

message Model {
    string id = 1;
    string name = 2;
    string email = 3;
    string phone = 4;

    message Address {
        string  street = 1;
        string  city = 2;
        string state = 3;
        string country = 4;
    }

    repeated Address address = 5;
}

message GetRequest {
    string keyword = 1;
}

message FindRequest {
    string id = 1;
}

message CreateRequest {
    Model user = 1;
}

message GetResponse {
    string status = 1;
    repeated Model users = 2;
}

message FindResponse {
    string status = 1;
    Model user = 2;
}

message CreateResponse {
    string status = 1;
    string message = 2;
}
```
Mình đã khởi tạo 1 file proto để giao tiếp giữa 2 instance với nhau. Instance A sẽ request trực tiếp đến method của instance B và nhận được response giống với structure đã được định dạng sẵn trong file proto. Tuy nhiên file này vẫn chưa chạy được nhiệm vụ tiếp theo bạn phải generate nó ra với language bạn sử dụng. ở đây mình sử dụng Go nên để generate nó mình làm như sau

### Generate file proto

Các bạn có thể tham khảo thêm generate với ngôn ngữ khác tại [https://grpc.io/docs/](https://grpc.io/docs/)
```sh
$ protoc -I user/ user/user.proto --go_out=plugins=grpc:user
```
Việc này sẽ tạo ra 1 file go với tên user.pb.go các bạn không cần quan tâm nhiều đến file này mọi thứ đã được định nghĩa ở file nguồn user.proto
Công việc còn lại là làm thế nào để sử dụng file này giao tiếp giữa 2 instance

### Cấu hình trong instance server
Trong instance server các bạn khai báo như sau

Import file `user.pb.go` vào go và khởi tạo struct `server`
```go
import (
    pb "github.com/dung13890/micro-go/pb/user"
    "golang.org/x/net/context"
    "google.golang.org/grpc"
)

type server struct {
    request []*pb.GetRequest
}
```

Tiếp tục khai báo method trong instance của server sử dụng `user.pb.go`

```go
func (s *server) CreateUser(ctx context.Context, req *pb.CreateRequest) (*pb.CreateResponse, error) {
    return &pb.CreateResponse{}, nil
}
```
Như vậy tất cả request và response đều được định dạng format trước ở trong file`user.pb.go` không cần biết các instance bạn sử dụng ngôn ngữ gì và cấu trúc code của bạn ra sao thì các format request & response vẫn mãi ko bị thay đổi

Hãy mở một port để kết nối cho phép các instance khác giao tiếp đến nó
```go
func main() {
    lis, err := net.Listen("tcp", ":8080")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    srv := grpc.NewServer()
    pb.RegisterUserServer(srv, &server{})
    srv.Serve(lis)
}
```
### Cấu hình ở instance client
Ở instance client thì đơn giản hơn hãy mở 1 giao tiếp RPC đến instance server bằng cách
```go
func main() {
    conn, err := grpc.Dial("user:8080", grpc.WithInsecure())
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewUserClient(conn)
    request := &pb.GetRequest{
        Keyword: "params request",
    }
    resp, err := client.CreateUser(context.Background(), request)
    // To do something with resp from instance server response
}
```

## Kết luận
Như vậy chúng ta đã có thể giao tiếp giữa các instance với nhau mà không cần qua các APIs. Với gRPC sử dụng giao tiếp RPC và nó gọi trực tiếp đến các method của những instance khác nhau về hiệu năng thì vượt trội hơn rất nhiều so với APIs. Nếu bạn muốn sử dụng authentication gRPC vẫn hoàn toàn có thể đáp ứng tốt điều đó đảm bảo cho các instance được private. Hiện tại gRPC vẫn chưa được sử dụng phổ biến nhưng với kiến trúc Microservices trong tương lai thì nó sẽ hữu dụng rất nhiều. Bài viết này sẽ hữu ích cho các bạn thích thú xây dựng với những ứng dụng theo hướng microservices. Chúc các bạn thành công.

## Tham khảo
Tài liệu grpc official: [https://grpc.io/docs/quickstart/go.html](https://grpc.io/docs/quickstart/go.html)\
Repo Project: [https://github.com/dung13890/micro-go](https://github.com/dung13890/micro-go)\
Bài viết tham khảo: [Building High Performance APIs In Go Using gRPC And Protocol Buffers](https://medium.com/@shijuvar/building-high-performance-apis-in-go-using-grpc-and-protocol-buffers-2eda5b80771b)\
