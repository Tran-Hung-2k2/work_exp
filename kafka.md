# Kafka KRaft

## Core Concept

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
- Metadata chứa: cluster info (clusterId, quorum), broker (id, host, trạng thái), topic (name, config), partition (leader, ISR, replica, epoch), ACL/security, config, feature/version

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

- Controller tạo snapshot định kỳ (tại offset N) để capture full metadata state → giảm chi phí replay log
- Snapshot lưu state đã materialize (topics, partitions, ISR, config, ACL, …) tại thời điểm đó
- Restart / catch-up:
    - load snapshot gần nhất (offset N)
    - fetch metadata log từ N+1 → latest
    - replay log → rebuild state hiện tại
- Snapshot + log = source of truth (snapshot chỉ là checkpoint, log vẫn authoritative)
- Compaction:
    - log cũ trước snapshot có thể được truncate
    - giúp giảm disk + tăng tốc recovery
- Broker cũng dùng cơ chế tương tự:
    - load snapshot → replay → build MetadataCache

## ZooKeeper vs KRaft

- ZooKeeper: metadata = in-memory tree; disk = backup (log + snapshot); write = random mutation
- KRaft: metadata = append-only log; disk = source of truth; write = sequential append

# Kafka Operations / Production Questions

## Cluster / Consensus (KRaft)

- Nếu cluster bị network partition thì KRaft xử lý thế nào?
    - nhóm có majority controller → tiếp tục hoạt động
    - nhóm thiểu số → không commit metadata (read-only hoặc fail)

- Nếu controller leader chết thì chuyện gì xảy ra?
    - controller khác trong quorum elect leader mới (Raft election)
    - metadata log vẫn đảm bảo consistency

- Bao nhiêu controller là hợp lý?
    - 3 hoặc 5 (odd number) để đảm bảo majority quorum

- Nếu mất majority controller?
    - cluster không thể commit metadata → gần như “freeze control plane”

- Split brain có xảy ra không?
    - không, vì chỉ majority mới commit

## Partition / Topic

- Có thể tăng partition không?
    - có → online
    - nhưng:
        - ordering theo key có thể bị phá
        - key hash mapping thay đổi

- Có thể giảm partition không?
    - không (Kafka không support shrink partition)

- Partition mất leader thì sao?
    - controller elect leader mới từ ISR

- Nếu ISR rỗng?
    - tuỳ config:
        - unclean leader election → có thể mất data
        - hoặc block write

- Rebalance partition có ảnh hưởng gì?
    - network spike
    - disk I/O tăng
    - latency tăng

## Replication / Durability

- acks=0 / 1 / all khác gì?
    - 0: fire-and-forget (mất data dễ)
    - 1: leader ack
    - all: majority ISR ack → an toàn nhất

- Khi follower lag quá xa thì sao?
    - bị kick khỏi ISR

- ISR là gì?
    - tập follower “in-sync” với leader

- Leader commit khi nào?
    - khi đủ ack (theo config, thường = all → ISR)

- High watermark là gì?
    - offset cao nhất đã commit (safe để read)

## Data loss / Failure

- Khi nào Kafka mất data?
    - acks != all
    - unclean leader election bật
    - replication factor thấp

- Nếu leader crash trước khi replicate?
    - message chưa commit → mất

- Nếu disk full thì sao?
    - broker stop accept write
    - cluster có thể degrade

## Consumer

- Consumer group hoạt động thế nào?
    - mỗi partition chỉ assign cho 1 consumer trong group

- Rebalance xảy ra khi nào?
    - consumer join/leave
    - topic/partition thay đổi

- Rebalance có vấn đề gì?
    - pause consumption
    - latency spike

- Offset commit lưu ở đâu?
    - \_\_consumer_offsets topic

- Auto commit vs manual commit?
    - auto: dễ dùng nhưng risk duplicate
    - manual: control tốt hơn

## Performance / Scaling

- Kafka scale theo gì?
    - partition (đơn vị parallelism)

- 1 partition chịu được bao nhiêu throughput?
    - phụ thuộc disk + network (thường MB/s)

- Tăng throughput bằng cách nào?
    - tăng partition
    - tăng batch size
    - tăng compression

- Bottleneck thường ở đâu?
    - disk I/O
    - network
    - controller (metadata ops nhiều)

## Storage / Retention

- Retention hoạt động thế nào?
    - delete theo time hoặc size

- Log compaction dùng khi nào?
    - giữ latest value theo key (event sourcing)

- Segment là gì?
    - file nhỏ trong log → giúp delete nhanh

- Có reclaim space ngay khi delete không?
    - không, phụ thuộc segment rolling

## Metadata / KRaft

- Metadata lưu ở đâu?
    - \_\_cluster_metadata (log)

- Broker lấy metadata như thế nào?
    - fetch log → replay → build state RAM

- Metadata có bị stale không?
    - có thể lag nếu broker chưa catch up

- Snapshot để làm gì?
    - giảm thời gian replay log

## Networking / Failure scenarios

- Nếu broker bị partition khỏi cluster?
    - bị remove khỏi ISR
    - leader có thể bị chuyển

- Nếu client connect tới broker cũ?
    - broker redirect metadata

- Nếu DNS / LB fail?
    - client retry broker khác

---

## Security / Config

- Kafka có hỗ trợ auth không?
    - SASL, SSL

- ACL hoạt động thế nào?
    - kiểm soát read/write per topic/group

---

## Monitoring / Debug

- Theo dõi gì?
    - consumer lag
    - ISR size
    - request latency
    - disk usage

- Tool debug?
    - kafka-consumer-groups
    - kafka-topics
    - JMX metrics

- Khi Kafka chậm thì check gì?
    - disk I/O
    - GC
    - network
    - partition skew

---

## TL;DR

- Kafka fail chủ yếu ở:
    - replication / ISR
    - controller quorum
    - disk / network

- Scale bằng:
    - partition

- Consistency dựa vào:
    - leader + replication + ack
