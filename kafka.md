## Core concept

- Kafka = distributed append-only log system; Data model: topic → partition → offset; mỗi partition = 1 log tuần tự

## Storage (cách lưu data)

- Data ghi append-only vào disk (sequential I/O); không update in-place; file chia thành segment; OS page cache dùng để cache read

## Partition & replication

- Mỗi partition có: 1 leader, N follower; leader xử lý read/write; follower replicate từ leader

## Write flow (data)

1. Producer gửi record tới leader
2. Leader append vào log
3. Replicate tới follower
4. Follower ACK
5. Leader commit (acks=1/all)

## Read flow

- Consumer đọc theo offset; pull model (consumer fetch); offset = position trong log

## Retention

- Xoá theo: time (retention.ms), size (retention.bytes); log compaction: giữ latest value theo key

## KRaft (metadata management)

- Kafka tự quản lý metadata (không dùng ZooKeeper); dùng Raft consensus

## Metadata storage

- Metadata lưu trong topic: \_\_cluster_metadata; 1 partition duy nhất chứa toàn bộ state; mỗi thay đổi = 1 record append vào log

## Controller quorum

- Nhóm controller node dùng Raft; 1 leader (active controller); commit khi majority ACK

## Write flow (metadata)

1. Broker gửi request (create topic, update…)
2. Controller leader nhận
3. Append vào metadata log
4. Replicate tới quorum
5. Majority ACK → commit
6. Broker fetch metadata mới

## Metadata state (runtime)

- Broker: replicate metadata log, replay → build state trong RAM; RAM = MetadataCache (state hiện tại); query metadata → đọc RAM (không đọc log)

## Snapshot & recovery

- Controller tạo snapshot định kỳ; restart: load snapshot, replay log từ offset

## ZooKeeper vs KRaft

### ZooKeeper

- metadata = in-memory tree; disk = backup (log + snapshot); write = random mutation

### KRaft

- metadata = append-only log; disk = source of truth; write = sequential append
