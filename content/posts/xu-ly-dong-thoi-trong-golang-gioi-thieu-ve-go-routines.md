---
title: "Xử lý đồng thời trong Golang giới thiệu về Go routines"
date: 2018-01-01T16:04:10Z
tags: ["concurrency", "routines", "Golang"]
description: "Hiện nay có rất nhiều những ngôn ngữ lập trình hỗ trợ xử lý đồng thời (Concurrency) hoặc multiple threed. Công việc này vừa mang lại hiệu năng về tốc độ đồng thời có thể tận dụng hết được tài nguyên của phần cứng"
---

Hiện nay có rất nhiều những ngôn ngữ lập trình hỗ trợ xử lý đồng thời (Concurrency) hoặc multiple threed. Công việc này vừa mang lại hiệu năng về tốc độ đồng thời có thể tận dụng hết được tài nguyên của phần cứng. Trong Go cũng vậy việc tách nhỏ 1 task lớn ra thành nhiều task con xử lý đồng thời sẽ mang lại 1 hiệu năng đáng kể

![golang](https://viblo.asia/uploads/e01153c6-306a-44cd-92cc-a95d3e24839b.jpg)
# Giới thiệu

Golang là một ngôn ngữ lập trình khá thú vị khi mới tiếp cận bạn sẽ thấy ngỡ ngàng vì lối tư duy của Go khác hoàn toàn với những ngôn ngữ như `PHP`, `Node`, `ruby` hoặc `java`. Để  code được go các bạn sẽ phải  tập làm quen với các khái niệm cơ bản như `kiểu dữ liệu`, `hàm`, `con trỏ` v..v...

## Giải bài toán fibonacci

Để làm quen xử lý đồng thời mình sẽ đưa ra 1 ví dụ minh họa cụ thể về giải bài toán fibonacci

*Như những ngôn ngữ khác cách thuật toán cho bài toán fibonacci khá đơn giản*

```go
package main

import (
    "fmt"
)

func main() {
    x := 0
    y := 1
    array := []int{}
    for x < 1000 {
        array = append(array, x)
        y = x + y
        x = y - x
    }
    fmt.Printf("%v\n", array)
}
// Output
[0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987]
```
*Với cách hoạt động xử lý đồng thời có thể rút gọn lại như sau*

```go
package main

import (
    "fmt"
)

func main() {
    x := 0
    y := 1
    array := []int{}
    for x < 1000 {
        array = append(array, x)
        x, y = y, x+y
    }
    fmt.Printf("%v\n", array)
}
// Output
[0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987]
```
Bài toán trên đã được xử lý ngắn gọn hơn chỉ với 1 dòng code `x, y = y, x+y`
Ở đây tức là với mỗi lần lặp 2 tác vụ `x = y` và `y = x+y` được xử lý cùng 1 lúc có thể hiểu rằng
`x1 = y0` và `y1 = x0+y0` 2 câu lệnh này diễn ra cùng nhau.

## Goroutines
Một khái niệm nữa mình muốn giới thiệu đó chính là goroutines
**Goroutine** được định nghĩa là 1 hàm mà có thể chạy đồng thời với những hàm khác. Cú pháp của nó rất đơn giản chỉ cần thêm từ khóa go vào trước khi gọi hàm
```go
go func () {
 To do something...
}()
```
Một Goroutines chạy tốn rất ít tài nguyên chỉ tốn khoảng 2Kb trong stack và khi chạy xong sẽ bị hủy bởi runtime

*1 Example với goroutines*
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    array := []int{}
    chunk1 := 0
    chunk2 := 0
    for i := 0; i < 20; i++ {
        array = append(array, i)
    }
    fmt.Printf("%v\n", array)
    go func() {
        for i := 0; i < 10; i++ {
            chunk1 += array[i]
        }
        fmt.Printf("chunk1= %v\n", chunk1)
    }()
    go func() {
        for i := 10; i < 20; i++ {
            chunk2 += array[i]
        }
        fmt.Printf("chunk2= %v\n", chunk2)
    }()

    fmt.Printf("chunk1 + chunk2= %v\n", chunk1+chunk2)
    time.Sleep(2 * time.Second)
}
// Output
[0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19]
chunk1 + chunk2= 0
chunk1= 45
chunk2= 145
```
> Với bài toán trên mình có mảng `[0,1, ...19]`. Mình muốn tính tổng các phần tử trong mảng đó. Ý tưởng mình nghĩ ra là chia nhỏ mảng đó ra thành 2 mảng con và thực hiện tính toán đồng thời. `[0,1, ...9]` và `[10,11,...19]`
Khi xem output kết quả lại không được như mình mong đợi giá trị total lại bằng **(0)** (false)

Tìm ra nguyên nhân thì rõ ràng đoạn code trên chạy không đợi chờ nhau.
> Khi câu lệnh `fmt.Printf("chunk1 + chunk2= %v\n", chunk1+chunk2)` được thực thi đồng nghĩa với việc 2 goroutine (chunk1) và (chunk2) cũng thực thi theo và nó ko đợi công việc của 2 goroutine kết thúc đã in luôn giá trị và kết quả = 0 ;(
## WaitGroup
Tìm qua 1 số tài liệu và hiểu được nguyên nhân mình đã tìm ra giải pháp để xử lý vấn đề này.

*WaitGroup sẽ chờ một tập hợp goroutines kết thúc. Hàm goroutines chính sẽ thêm số goroutines mà nó muốn chờ, mỗi hàm goroutine khi chạy xong sẽ gọi Done(). Cho tới khi mà các goroutines chưa được chạy xong, thì waitgroup sẽ block chương trình tại thời điểm đó.*

Mình sẽ chỉnh sửa lại đoạn code trên với WaitGroup
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    array := []int{}
    chunk1 := 0
    chunk2 := 0
    for i := 0; i < 20; i++ {
        array = append(array, i)
    }
    var wg sync.WaitGroup
    wg.Add(2)
    fmt.Printf("%v\n", array)
    go func() {
        defer wg.Done()
        for i := 0; i < 10; i++ {
            chunk1 += array[i]
        }
        fmt.Printf("chunk1= %v\n", chunk1)
    }()
    go func() {
        defer wg.Done()
        for i := 10; i < 20; i++ {
            chunk2 += array[i]
        }
        fmt.Printf("chunk2= %v\n", chunk2)
    }()
    wg.Wait()
    fmt.Printf("chunk1 + chunk2= %v\n", chunk1+chunk2)
    time.Sleep(2 * time.Second)
}
// Output
[0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19]
chunk2= 145
chunk1= 45
chunk1 + chunk2= 190
```
> **Great!** Đúng như kết quả mình mong đợi 190 là tổng từ `0 ~ 19 = 190` và nó được tách nhỏ bằng tổng của 2 kết quả `0 ~ 10` + `10 ~ 19`

Như vậy có thể kết luận tất cả các hàm ở bên dưới `wg.Wait()` chỉ có thể thực thi khi `wg` done hết.
Mỗi hàm trong `wg.Done()` trong goroutines sẽ thực thi khi goroutines chạy xong. Mỗi lần `wg.Done()` sẽ giảm `wg` đi 1 và cho tới khi `wg` về 0 `wg.Wait()` mới bắt đầu cho phép chức năng chạy xuống bên dưới.

## Conclusion
Golang còn rất nhiều những điều thú vị, nó có tốc độ thực thi cực nhanh và giải quyết được những vấn đề phức tạp. Chính bởi vậy mà ngay nay các các hệ thống services đang chuyển dần sang Go.
Trong bài này mình chỉ giới thiệu qua về Concurrency In golang. Thay vì phải giải quyết 1 task lớn chúng ta hay chia nhỏ chúng ra và xử lý đồng thời. Vấn đề về performance chắc chắn sẽ được giải quyết.
> Bài sau mình sẽ giới thiệu về `Channel`, `Channel` sẽ là 1 công cụ rất cần thiết và tuyệt vời cho những vấn đề  concurrency này.
