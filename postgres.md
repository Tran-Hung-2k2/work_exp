# PostgreSQL

## Storage (cách lưu data)

- Data lưu theo **page (block)**, default: **8KB/page**
- 1 table = 1 hoặc nhiều file (segment ~1GB/file)
- Mỗi file gồm nhiều page → PostgreSQL đọc/ghi **theo page**, không theo row
- Một page gồm:
    - **Page Header** (metadata)
    - **Item Pointer Array (line pointers)**:
        - trỏ đến vị trí tuple trong page
        - giúp tránh phải di chuyển pointer khi tuple thay đổi
    - **Tuple Data (row versions)**

- Tuple không nằm liên tục logic, mà được truy cập qua pointer giúp hỗ trợ MVCC và update hiệu quả

## Buffer Manager (Shared Buffers)

- Shared buffers = **RAM cache của PostgreSQL**
- Khi đọc/ghi:
    - Load page từ disk → shared buffers
    - Mọi thao tác diễn ra trên RAM trước

### Trạng thái page:

- **Clean page**: giống disk
- **Dirty page**: đã bị modify → cần flush xuống disk

### Cơ chế:

- LRU-like (Least Recently Used) eviction: xóa page ít dùng nhất khi cần chỗ cho page mới
- Buffer pinning (tránh flush page khi đang được sử dụng) + locking để tránh race condition

## MVCC (Multi-Version Concurrency Control)

- PostgreSQL dùng **MVCC = không overwrite data**
- UPDATE/DELETE: không sửa trực tiếp mà tạo **tuple version mới** -> version cũ vẫn tồn tại

### Cấu trúc tuple (row version)

Mỗi tuple có metadata:

- `xmin`: transaction tạo
- `xmax`: transaction xoá/update
- `ctid`: pointer tới version mới (nếu update)

### Visibility (cực quan trọng)

Mỗi transaction có **snapshot** gồm:

- danh sách transaction đang active
- xmin/xmax boundary

👉 Khi đọc:

- PostgreSQL kiểm tra:
    - xmin đã commit chưa?
    - xmax có xoá chưa?

→ quyết định tuple có **visible** hay không

### Dead tuple

- Tuple cũ sau UPDATE/DELETE → trở thành **dead**
- Không bị xoá ngay vì:
    - có thể transaction khác vẫn cần

👉 VACUUM sẽ:

- reclaim space
- cập nhật visibility map

## WAL (Write-Ahead Log)

- WAL = **log tuần tự (append-only)** của mọi thay đổi
- Lưu trong `pg_wal/`

### Nguyên tắc:

> **WAL phải được flush xuống disk trước data page**

### WAL record chứa gì?

Không phải full data (trừ trường hợp đặc biệt), mà là:

- loại operation (INSERT/UPDATE/DELETE)
- block id (page nào)
- offset (tuple nào)
- dữ liệu thay đổi (delta)

### LSN (Log Sequence Number)

- Mỗi WAL record có 1 **LSN**. Mỗi data page cũng lưu LSN
- Page chỉ được flush khi: page.LSN <= flushed WAL LSN

### WAL Buffer

- WAL ghi vào **WAL buffer (RAM)** trước
- Sau đó mới flush xuống disk

### Group Commit

- Nhiều transaction COMMIT gần nhau:
- dùng chung 1 lần fsync
  → giảm chi phí I/O

### Checkpoint:

- Checkpoint = thời điểm PostgreSQL đảm bảo mọi dirty page trong RAM đã được ghi xuống disk
- Sau checkpoint → WAL cũ không còn cần thiết nữa -> WAL segment trước checkpoint có thể remove hoặc recycle -> đảm bảo WAL không phình quá lớn

## Tổng quan pipeline

### Critical path (COMMIT nhanh):

Modify → WAL → fsync WAL → DONE

### Background (chậm nhưng async):

Dirty pages → checkpoint → disk (random write)

### Write flow

1. Modify data page trong shared buffer (RAM)
2. Tạo WAL record (ghi thay đổi)
3. Append vào WAL buffer
4. COMMIT → fsync WAL xuống disk
5. Data page flush xuống disk sau (async, checkpoint)

#### Crash recovery

- Crash → đọc WAL từ checkpoint
- Replay lại thay đổi → restore state đúng

#### ACID trong PostgreSQL

##### Atomicity

- Transaction commit/rollback; nếu crash trước commit → bỏ toàn bộ; nếu commit → redo từ WAL

##### Consistency

- Constraint: PK, FK, UNIQUE, CHECK; Trigger / rule đảm bảo data hợp lệ; MVCC đảm bảo không thấy trạng thái “nửa commit”

##### Isolation

- Dựa trên MVCC + snapshot; mỗi transaction thấy version data tại thời điểm bắt đầu
- Tránh: dirty read, non-repeatable read (tuỳ isolation level)
- Write conflict được serialize bằng row-level lock
- Isolation level:
    - Read Uncommitted:
        - cho phép dirty read (nhưng PostgreSQL thực tế = Read Committed)
    - Read Committed (default):
        - chỉ đọc data đã commit
        - có thể xảy ra non-repeatable read, phantom read
    - Repeatable Read:
        - snapshot tại start transaction
        - không có dirty read, non-repeatable read
        - (Postgres) gần như không có phantom read
    - Serializable:
        - strongest isolation
        - đảm bảo như chạy tuần tự (serial execution)
        - có thể bị rollback (serialization failure)

##### Durability

- COMMIT = WAL fsync thành công; crash → replay WAL → không mất data đã commit; full page write giúp tránh corrupted page khi crash

## TOAST (The Oversized-Attribute Storage Technique)

### 1. Vấn đề TOAST giải quyết

PostgreSQL lưu data theo **page (8KB)**:

- 1 tuple phải nằm **trọn trong 1 page**
- Nếu row quá lớn → không thể fit vào page

👉 Ví dụ:

- TEXT / JSONB vài KB – vài MB
- BYTEA (binary)

### 2. Ý tưởng của TOAST

> Tự động tách (offload) dữ liệu lớn ra khỏi main table

### 3. Cách TOAST hoạt động

Khi 1 giá trị quá lớn:

#### Step 1: thử nén (compression)

- PostgreSQL dùng:
    - `pglz` (default)
- Nếu nén đủ nhỏ:
    - ✔ giữ lại inline trong page

#### Step 2: nếu vẫn quá lớn → TOAST

- dữ liệu bị:
    - cắt thành nhiều chunk (~2KB mỗi chunk)
- lưu vào:
    > **TOAST table riêng**

#### Step 3: tuple chính chỉ giữ pointer

- pointer gồm:
    - OID của TOAST table
    - list chunk id (các chunk chứa dữ liệu)

## Query planner / optimizer

## Parallel query

## Vacuum / Autovacuum

Làm sạch như thế nào?

## Replication

Streaming replication (WAL-based)
primary → standby
read scaling
Synchronous replication
commit phải chờ replica confirm
Logical replication
replicate theo table / row

## Index

- B-Tree; Hash Index; GIN (inverted index); GiST (tree tổng quát); BRIN; Covering Index

## Partition (PostgreSQL)

### 1. Khái niệm

- Chia 1 table lớn thành nhiều partition nhỏ
- Vẫn query như 1 table

### 2. Loại

- RANGE (time-based)
- LIST (category)
- HASH (even distribution)

### 3. Cách hoạt động

- INSERT:
    - route vào partition theo key

- SELECT:
    - dùng partition pruning để scan ít data

### 4. Ưu điểm

- Tăng performance query
- Giảm scan I/O
- Dễ delete (drop partition)
- Vacuum nhẹ hơn

### 5. Nhược điểm

- Không có global index
- Unique constraint khó
- Quá nhiều partition → chậm planner

### 6. Use case

- Time-series data
- Log / event
- Data lifecycle management

### 7. Insight

- Partition = tối ưu query (reduce scan)
- Không phải scaling multi-node

## Sharding

### 1. Khái niệm

- Sharding = chia database lớn thành nhiều shard nhỏ trên nhiều server
- Mỗi shard chứa subset dữ liệu (theo row)
- Là dạng horizontal partitioning

### 2. Mục tiêu

- Scale horizontally
- Giảm load trên 1 server
- Tăng throughput

### 3. Cách hoạt động

- Dùng shard key để xác định shard
- Router/app:
    - tính shard key
    - gửi query đúng shard

### 4. Sharding strategies

- Range-based
- Hash-based
- Geo-based

### 5. Ưu điểm

- Scale tốt (thêm server)
- Query nhanh hơn (data nhỏ hơn)
- Fault isolation

### 6. Nhược điểm

- Cross-shard query phức tạp
- Distributed transaction khó
- Rebalancing khó
- Không enforce global constraint dễ

### 7. So với partition

- Partition: trong 1 DB
- Sharding: nhiều DB (distributed)

### 8. Khi dùng

- Data rất lớn (TB+)
- Traffic cao
- 1 server không đủ

## Connection pooling

## TimescaleDB (time-series)

## PostgreSQL Backup

### 1. Logical backup

- Tool: pg_dump, pg_dumpall
- Backup dạng SQL
- Ưu:
    - flexible, portable
- Nhược:
    - chậm với DB lớn
    - không PITR

### 2. Physical backup

- Tool: pg_basebackup
- Copy data directory (binary)
- Ưu:
    - nhanh
    - phù hợp DB lớn
    - hỗ trợ PITR (khi có WAL)
- Nhược:
    - không selective

### 3. Continuous backup (WAL)

- Base backup + WAL archive
- Cho phép:
    - Point-in-time recovery

### 4. Backup strategy

- Small DB:
    - pg_dump

- Production:
    - pg_basebackup + WAL

## Patroni

## Paging

### Offset pagination

- support jump page
- O(n) với offset lớn (scan + skip) -> chậm; duplicate / missing khi data thay đổi

### Cursor pagination

- O(1) (index seek); stable hơn offset; phù hợp infinite scroll / real-time feed
- không jump page; có thể miss data mới insert (tuỳ direction)

### Count

1. COUNT(\*) = scan toàn bộ table (O(n)) (do MVCC → phải check visibility từng row)
2. reltuples (estimate)
    - O(1), cực nhanh
    - phải ANALYZE để update; nhớ filter relkind='r'

# Interview Questions

1. INNER JOIN vs LEFT JOIN khác gì?
2. GROUP BY hoạt động như thế nào?
3. HAVING vs WHERE khác gì?
4. COUNT(\*) vs COUNT(column) khác gì?
5. NULL xử lý như thế nào? (COALESCE, IS NULL)
6. Viết query tìm top N record theo group
7. Tìm duplicate record trong table
8. DELETE duplicate nhưng giữ lại 1 record
9. Self join dùng khi nào?
10. Subquery vs JOIN khác gì?
11. ROW_NUMBER vs RANK vs DENSE_RANK khác gì?
12. Viết query lấy record mới nhất mỗi group
13. Tính running total
14. LAG / LEAD dùng khi nào?
15. Query chậm → debug như thế nào?
16. Index hoạt động ra sao?
17. Khi nào index không được dùng?
18. EXPLAIN ANALYZE đọc như thế nào?
19. Seq scan vs index scan khác gì?
20. MVCC là gì?
21. Isolation level khác nhau thế nào?
22. Deadlock là gì? xử lý sao?
23. Transaction ACID là gì?
24. WAL hoạt động ra sao?
25. Design DB cho e-commerce
26. Design hệ thống log / time-series
27. Khi nào nên denormalize?
28. Partition table khi nào?
29. Index strategy cho table lớn?
30. Tại sao COUNT(\*) chậm trong PostgreSQL?
31. OFFSET pagination vs cursor pagination?
32. Làm sao paginate 10M rows?
33. Index nào cho JSONB?
34. Tại sao index vẫn không được dùng?
