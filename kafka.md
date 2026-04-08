# Kafka KRaft

## Core Concept

- Kafka gồm control plane (metadata management) và data plane (message storage/replication). Control plane dùng KRaft (Raft consensus) để quản lý metadata; data plane dùng log-based storage để lưu message.
- Kafka = distributed append-only log system; Data model: topic → partition → offset; mỗi partition = 1 log tuần tự

## Storage (cách lưu data)

- Data ghi append-only vào disk (sequential I/O); không update in-place; file chia thành segment; OS page cache dùng để cache read
- Segment là đơn vị vật lý của một partition. Mỗi partition được chia thành nhiều segment file để tối ưu lưu trữ, xóa dữ liệu và truy xuất
  Mỗi segment gồm 3 file chính:
- `.log`: chứa dữ liệu message thật (append-only log)
- `.index`: map offset → vị trí byte trong file .log, giúp nhảy nhanh đến vị trí dữ liệu theo offset thay vì scan toàn file
- `.timeindex`: map timestamp → offset gần nhất, giúp tìm offset theo thời gian (time-based lookup)

- Lợi ích của segment:
    - Xóa dữ liệu nhanh (retention xóa theo segment, không xóa từng message)
    - Tăng hiệu năng đọc (index + segment giúp random access nhanh)
    - Tăng hiệu quả recovery khi restart broker
    - Giảm chi phí quản lý file lớn (tránh 1 file log khổng lồ)

## Partition & replication

- Partition là đơn vị nhỏ nhất của parallelism trong Kafka topic. Kafka chỉ đảm bảo ordering trong một partition, không đảm bảo giữa các partition.
- Mỗi partition có: 1 leader, N follower. Leader xử lý read/write (produce + fetch request). Follower replicate dữ liệu từ leader theo cơ chế append-only log

- ISR (In-Sync Replicas) là tập các replica còn đồng bộ với leader, tức là không bị lag quá ngưỡng (ví dụ replica.lag.time.max.ms hoặc cơ chế kiểm tra lag tương đương). Chỉ các replica nằm trong ISR mới được dùng để đảm bảo durability
- High Watermark là offset cao nhất đã được commit an toàn; consumer chỉ đọc tới mức này để đảm bảo consistency
- Failure handling: leader crash → controller chọn leader mới từ ISR; follower lag quá xa → bị loại khỏi ISR; nếu ISR rỗng thì có thể block ghi (an toàn) hoặc cho phép unclean leader election (có nguy cơ mất data)

- Lợi ích của partition:
    - Tăng throughput write: nhiều partition → nhiều leader → nhiều luồng ghi song song
    - Tăng throughput read: consumer group có thể scale ngang bằng cách tăng số consumer (mỗi consumer xử lý 1 hoặc nhiều partition)
    - Cho phép parallel processing theo key ordering:

- Producer gửi record có key -> Kafka xác định partition bằng hash `partition = hash(key) % num_partitions`. Cùng key luôn vào cùng partition, nên giữ ordering theo key. Không có key thì dùng sticky hoặc round-robin partitioning để cân bằng tải.
- Thay đổi số partition có thể làm đổi mapping, từ đó phá ordering theo key.

## Write flow (data)

1. Producer gửi record tới leader
2. Leader append vào log
3. Replicate tới follower
4. Follower ACK
5. Leader commit (acks=1/all)

## Read flow

- Consumer hoạt động theo pull model: consumer chủ động fetch dữ liệu từ broker (Consumer tự quyết định batch size, tốc độ consume và backpressure.), khác với push model (như RabbitMQ), nơi broker đẩy dữ liệu tới consumer.

1. Consumer gửi fetch request (topic, partition, offset)
2. Broker (leader) đọc data từ log (disk hoặc page cache)
3. Trả batch record cho consumer
4. Consumer xử lý và update offset (local hoặc commit)

- Offset là index tuần tự cho record trong partition (0 → n), consumer dùng offset để resume khi restart
- Consumer group giúp scale parallel consumption, mỗi partition chỉ được assign cho 1 consumer trong group

- Read consistency: consumer chỉ đọc được data đã commit (đã replicated); data chưa commit có thể bị mất nếu leader crash

## Retention

- Có 3 cơ chế:
    - Time-based (retention.ms): xoá data theo thời gian
    - Size-based (retention.bytes): xoá khi vượt dung lượng
    - Log compaction: giữ latest value theo key (không xoá theo time/size)

- Retention (time/size)
    - Kafka xoá theo segment (không xoá từng record)
    - Khi vượt threshold → xoá segment cũ

- Log compaction
    - Giữ lại record mới nhất theo key, xoá các record cũ cùng key
    - Chạy background (log cleaner)
    - Không xảy ra ngay (eventual cleanup)
    - Dùng cho:
        - state store
        - \_\_consumer_offsets
        - changelog/event sourcing

## KRaft (metadata management)

- Kafka tự quản lý metadata (không dùng ZooKeeper); dùng Raft consensus; 1 leader (active controller)

## Metadata storage

### Control plane metadata

- Metadata lưu trong topic: \_\_cluster_metadata; 1 partition duy nhất chứa toàn bộ state; mỗi thay đổi = 1 record append vào log
- Metadata chứa: cluster info (clusterId, quorum), broker (id, host, trạng thái), topic (name, config), partition (leader, ISR, replica, epoch), ACL/security, config, feature/version
- Replication chỉ giữa controller (Raft quorum). Broker fetch metadata mới từ controller và build state trong RAM

- Write flow (metadata):
    1. Broker gửi request (create topic, update…)
    2. Controller leader nhận
    3. Append vào metadata log
    4. Replicate tới quorum
    5. Majority ACK → commit
    6. Broker fetch metadata mới

### Data plane metadata

- Dùng log compaction (giữ offset mới nhất)
- Consumer metadata lưu trong topic: \_\_consumer_offsets; nhiều partition (scale theo consumer group); dùng log compaction (giữ offset mới nhất)
- Consumer metadata chứa group metadata: group.id, member, assignment, generation, offset: topic, partition, committed offset

- Write flow (consumer offset)
    1. Consumer commit offset
    2. Gửi request tới group coordinator (broker)
    3. Broker ghi vào \_\_consumer_offsets

## Metadata state (runtime)

- Broker: replicate metadata log, replay → build state trong RAM; RAM = MetadataCache (state hiện tại); query metadata → đọc RAM (không đọc log)

## Snapshot metadata & recovery

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
- Kraft tối ưu hơn vì append-only log tận dụng tốt disk; Raft đảm bảo consistency; không cần external system (ZooKeeper)

# Kafka Interview Questions

## Cluster / Consensus (KRaft)

- Nếu cluster bị network partition thì KRaft xử lý thế nào?
    - nhóm có majority controller → tiếp tục hoạt động
    - nhóm thiểu số → không commit metadata (read-only hoặc fail)
    - không có nhóm nào có majority controller → không bầu được controller leader → cluster không commit metadata → gần như “freeze control plane”. Data plane có thể vẫn chạy tạm thời nếu leader partition chưa bị ảnh hưởng, nhưng dễ degrade nhanh khi có failure

- Nếu controller leader chết thì chuyện gì xảy ra?
    - controller khác trong quorum elect leader mới (Raft election)
    - metadata log vẫn đảm bảo consistency

- Bao nhiêu controller là hợp lý?
    - Chọn 3 hoặc 5 controller vì Raft yêu cầu majority quorum, và số lẻ tối ưu hóa khả năng chịu lỗi mà không tạo ra tình huống split quorum (tie vote) như số chẵn. Nếu nhiều hơn 5 controller, overhead quản lý và latency có thể tăng lên do nhiều node phải đồng bộ metadata log.

- Nếu mất majority controller?
    - cluster không thể commit metadata → gần như “freeze control plane”

- Split brain có xảy ra không? (2 controller leader)
    - không, vì chỉ majority mới commit

## Partition / Topic

- Có thể tăng partition không?
    - có → online không downtime
    - nhưng ordering theo key có thể bị phá do key hash mapping thay đổi (partition = hash(key) % num_partitions), nên nếu num_partitions thay đổi thì key có thể map sang partition khác → ordering theo key bị phá

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

- Metadata có bị stale không? (bị cũ)
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

## Security / Config

- Kafka có hỗ trợ auth không?
    - SASL, SSL

- ACL hoạt động thế nào?
    - kiểm soát read/write per topic/group

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
