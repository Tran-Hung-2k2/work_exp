# PostgreSQL

- Storage (cách lưu data)
    - Data lưu theo page (default 8KB), 1 table = 1 hoặc nhiều file, mỗi file gồm nhiều page, DB đọc/ghi theo page
    - 1 page chứa nhiều tuple (row version), có header + pointer + data

- MVCC (multi-version)
    - Mỗi row có nhiều version (tuple), UPDATE/DELETE KHÔNG overwrite: → tạo row mới, row cũ vẫn tồn tại
    - Mỗi row có:
        - xmin: transaction tạo
        - xmax: transaction xoá/update
    - Query đọc theo snapshot → mỗi transaction thấy “view riêng”
    - Dead tuple sẽ được dọn bởi VACUUM

- WAL (Write-Ahead Log)
    - WAL = log tuần tự (append-only) của mọi thay đổi
    - Quy tắc: WAL phải ghi xuống disk TRƯỚC data page (write-ahead)

- Write flow
    1. Modify data trong shared buffer (RAM)
    2. Tạo WAL record (ghi thay đổi)
    3. Append vào WAL buffer
    4. COMMIT → fsync WAL xuống disk
    5. Data page flush xuống disk sau (async, checkpoint)

- Crash recovery
    - Crash → đọc WAL từ checkpoint
    - Replay lại thay đổi → restore state đúng

- ACID trong PostgreSQL
- Atomicity
    - Transaction commit/rollback; nếu crash trước commit → bỏ toàn bộ; nếu commit → redo từ WAL

- Consistency
    - Constraint: PK, FK, UNIQUE, CHECK; Trigger / rule đảm bảo data hợp lệ; MVCC đảm bảo không thấy trạng thái “nửa commit”

- Isolation
    - Dựa trên MVCC + snapshot; mỗi transaction thấy version data tại thời điểm bắt đầu
    - Tránh: dirty read, non-repeatable read (tuỳ isolation level)
    - Write conflict được serialize bằng row-level lock

- Durability
    - COMMIT = WAL fsync thành công; crash → replay WAL → không mất data đã commit; full page write giúp tránh corrupted page khi crash

# Index

- B-Tree; Hash Index; GIN (inverted index); GiST (tree tổng quát); BRIN; Covering Index

# Paging

- Offset pagination
    - support jump page
    * O(n) với offset lớn (scan + skip) -> chậm; duplicate / missing khi data thay đổi
- Cursor pagination
    - O(1) (index seek); stable hơn offset; phù hợp infinite scroll / real-time feed
    * không jump page; có thể miss data mới insert (tuỳ direction)
- Count:
    1. COUNT(\*) = scan toàn bộ table (O(n)) (do MVCC → phải check visibility từng row)
    2. reltuples (estimate)
    - O(1), cực nhanh
    * phải ANALYZE để update; nhớ filter relkind='r'

# Partition

# Backup DB

# Patroni

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
