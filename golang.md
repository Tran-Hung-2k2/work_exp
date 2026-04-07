# Go Notes

## DI (Dependency Injection)

- loose coupling qua interface; inject dependency qua constructor (DI thủ công) → phụ thuộc abstraction, không phụ thuộc implementation

## Interface & Embedding

- Go không có inheritance; implement đủ method → auto satisfy interface; embedding ≠ kế thừa; không có override → chỉ là method shadowing

## Web framework (Go)

- Fiber: nhanh, ít allocation, tối ưu performance
- Gin: chậm hơn chút nhưng ecosystem tốt hơn
- fasthttp: low-level, fastest nhưng khó dùng
- mux: đơn giản, gần với standard
- go-kit: hướng microservice

# Pattern

- Decorator: wrap behavior (middleware, logging, metric)
- Saga: transaction phân tán + rollback từng bước

# Concurrency (Go)

- goroutine nhẹ hơn thread OS; stack ~2KB, grow/shrink động; thread OS ~1MB–8MB cố định
- Go runtime quản lý (user-space), không qua kernel; context switch tại safe point → chỉ lưu PC, SP
- thread OS: interrupt bất kỳ → phải save full CPU context + syscall → chậm hơn
- M:N model: n goroutine chạy trên m thread
- race condition khi nhiều goroutine access shared data
- goroutine tạo/huỷ cực nhanh → scale hàng chục nghìn
