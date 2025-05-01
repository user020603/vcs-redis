# Báo cáo Redis 

## 1. Giới thiệu Redis

Redis (Remote Dictionary Server) là một hệ thống lưu trữ dữ liệu **mã nguồn mở, trong bộ nhớ (in memory)**, hoạt động dưới dạng **key-value store**. Tuy nhiên, Redis vượt trội hơn so với các key-value store khác nhờ hỗ trợ nhiều cấu trúc dữ liệu phức tạp. 

**Đặc điểm chính**

- **Tốc độ cao**: Do dữ liệu chủ yếu được lưu trữ và xử lý trên RAM, Redis cung cấp độ trễ đọc/ghi rất thấp (thường dưới milisecond)

- **Hỗ trợ nhiều cấu trúc dữ liệu**: Ngoài kiểu string cơ bản, Redis hỗ trợ Lists, Sets, Sorted Sets, Hashes, HyperLogLogs, Geospatial indexes, và Streams. Điều này cho phép giải quyết nhiều bài toán phức tạp một cách hiệu quả ngay tại tầng dữ liệu. 

- **Tính nguyên tử**: Các thao tác trên một cấu trúc dữ liệu đơn lẻ thường là nguyên tử, đảm bảo tính nhất quán dữ liệu. Redis cũng hỗ trợ Transactions và Lua scripting để thực thi nhiều lệnh một cách nguyên tử. 

- **Độ bền (Persistence)**: Mặc dù là in-memory, Redis cung cấp các cơ chế để lưu trữ dữ liệu trên disk (RDB, AOF), đảm bảo dữ liệu không bị mất khi khởi động lại server. 

- **Khả năng mở rộng (Scalability)** : Hỗ trợ Replication (Master-Slave) để tăng khả năng đọc và chịu lỗi, và Clustering để phân tán dữ liệu trên nhiều node, cho phép mở rộng cả ghi và đung lượng lưu trữ.

- **Tính linh hoạt**: Có thể được sử dụng như một database, cache, message broker, và nhiều vai trò khác. 

**Cơ chế hoạt động của Redis**

- **Lưu trữ dữ liệu trong RAM**: Đây là yếu tố cốt lõi tạo nên tốc độ của Redis. Mọi hoạt động đọc/ghi chính đều diễn ra trên bộ nhớ RAM. Dữ liệu có thể được lưu xuống disk một cách đồng bộ hoặc không đồng bộ (tùy cấu hình persistence) để đảm bảo độ bền. 

- **Xử lý request theo Event Loop (Single-Threaded)**: Redis sử dụng mô hình event loop dựa trên I/O multiplexing (như `epoll`, `kqueue`, `select`) để xử lý các kết nối và request từ client trên một luồng duy nhất (single thread). 

    - Khi một client kết nối hoặc gửi request, sự kiện đó được đưa vào hàng đợi sự kiện.
    - Event loop liên tục kiểm tra hàng đợi này.
    - Khi có sự kiện (ví dụ: dữ liệu đến từ socket), Redis xử lý lệnh tương ứng. Do các lệnh Redis thường rất nhanh (vì thao tác trên RAM), việc xử lý trên một luồng không gây tắc nghẽn đáng kể cho các request khác. 
    - Sau khi xử lý xong, kết quả được gửi trả lại client. 
    - Mô hình này tránh được overhead của việc tạo và quản lý nhiều luồng, đơn giản hóa việc xử lý đồng thời và tránh các vấn đề phức tạp liên quan đến race conditions khi truy cập dữ liệu chung. Tuy nhiên, các lệnh chạy lâu (như `KEYS` trên tập dữ liệu lớn hoặc các Lua script phức tạp) có thể block event loop, ảnh hưởng đến của các client khác. 

## 2. Persistence trong Redis (Độ bền dữ liệu)

Redis cung cấp các cơ chế để lưu trạng thái dữ liệu từ RAM xuống disk, đảm bảo dữ liệu không bị mất khi server gặp sự cố hoặc restart. 

**RDB (Redis Database Backup)**

- **Cơ chế**: Snapshotting (Chụp ảnh nhanh). Redis định kỳ tạo một bản sao (snapshot) của toàn bộ dataset trong bộ nhớ tại một thời điểm nhất định và lưu vào một file `.rdb` trên đĩa.
- **Cách hoạt động**: Redis `fork` một tiến trình con. Tiến trình con này sẽ thực hiện việc ghi dữ liệu vào file `.rdb` trong khi tiến trình cha tiếp tục phục vụ các request mới. Cơ chế copy-on-write của hệ điều hành giúp giảm thiểu ảnh hưởng hiệu năng lên tiến trình cha. 
- **Ưu điểm**: 
    - File `.rdb` nhỏ gọn, tối ưu cho việc backup và di chuyển. 
    - Tốc độ khôi phục từ file `.rdb` thường nhanh hơn AOF vì chỉ cần tải một file duy nhất. 
    - Ít ảnh hưởng đến hiệu năng ghi của Redis trong hoạt động bình thường (việc ghi xuống đĩa được thực hiện bởi tiến trình con).
- **Nhược điểm**: 
    - Có khả năng mất dữ liệu giữa các lần snapshot. Nếu Redis bị crasj trước khi snapshot tiếp theo được tạo, tất cả các thay đổi từ lần snapshot cuối cùng sẽ bị mất. Khoảng thời gian mất dữ liệu phụ thuộc vào tần suất cấu hình snapshot (ví dụ: `save 900 1` - lưu sau 900s nếu có ít nhất 1 key thay đổi). 

**AOF (Append Only File)**

- **Cơ chế**: Ghi lại log các lệnh ghi. Redis ghi lại tất cả các lệnh làm thay đổi dữ liệu (như `SET`, `INCR`, `LPUSH`, `SADD`, v.v.) vào một file `.aof` theo thứ tự chúng được thực thi. 
- **Cách hoạt động**: Khi Redis restart, nó sẽ đọc file `.aof` và thực thi lại tuần tự các lệnh đã ghiu để khôi phục lại trạng thái ban đầu. 
- **Chế độ ghi (fsync)**: 
    - `always`: Ghi vào disk sau mỗi lệnh (an toàn nhất nhưng lại chậm nhất).
    - `everysec` (mặc định): Ghi vào disk mỗi giây (cân bằng giữa an toàn và hiệu năng, có thể mất tối đa 1 giây dữ liệu)
    - `no`: Để hệ điều hành quyết định khi nào ghi (nhanh nhất nhưng kém an toàn nhất).
- **AOF Rewrite**: Do file `.aof` liên tục lớn lên, Redis cung cấp cơ chế "rewrite" để tạo ra một file `.aof` mới, nhỏ gọn hơn nhưng vẫn chứa đủ thông tin để tái tạo dataset hiện tại. Quá trình này cũng tương tự RDB, sử dụng tiến trình con. 
- **Ưu điểm**: 
    - Độ bền cao hơn RDB. Với cấu hình `everysec`, chỉ có thể mất tối đa 1 giây dữ liệu. Với `always`, gần như không mất dữ liệu (trừ trường hợp hệ thống file hoặc đĩa cứng gặp vấn đề).
    - File `.aof` dễ đọc và phân tích hơn (là log các lệnh).
- **Nhược điểm**: 
    - File `.aof` thường lớn hơn file `.rdb` tương đương.
    - Tốc độ khôi phục từng AOF có thể chậm hơn RDB, đặc biệt với file `.aof` lớn, vì phải thực thi lại từng lệnh. 
    - Có thể ảnh hưởng đến hiệu năng ghi nhiều hơn RDB (tùy thuộc vào `fsync`)

**Hybrid Persistence**

- **Cơ chế**: Kết hợp cả RDB và AOF. Khi bật chế độ này (từ Redis 4.0 trở đi), quá trình AOF rewrite sẽ không ghi lại từng lệnh mà ghi phần RDB snapshot vào đầu file AOF, sau đó tiếp tục ghi các lệnh thay đổi xảy ra trong quá trình rewrite.
- **Ưu điểm**: 
    - Tận dụng tốc độ khôi phục nhanh của RDB (phần snapshot ban đầu).
    - Đảm bảo độ bền cao của AOF (các lệnh thay đổi sau snapshot).
    - Cung cấp sự cân bằng tốt nhất giữa tốc độ khôi phục và độ an toàn dữ liệu. Đây thường là lựa chọn được khuyến nghị. 

## 3. Các kiểu dữ liệu trong Redis 

Redis nổi bật nhờ hỗ trợ nhiều loại cấu trúc dữ liệu, cho phép mô hình hóa hiệu quả hơn.

- **String**
    - Kiểu dữ liệu cơ bản nhất, có thể lưu trữ bất kỳ loại dữ liệu nào (text, số, dữ liệu nhị phân) lên đến 512MB.
    - Cấu trúc dữ liệu: Simple Dynamic String (SDS).
    - Sử dụng: Caching HTML fragments, lưu giá trị số (counters), bitmap operations.
    *   Lệnh chính: `SET`, `GET`, `DEL`, `INCR`, `DECR`, `MGET`, `MSET`, `APPEND`, `GETRANGE`, `SETRANGE`, `GETBIT`, `SETBIT`, `BITCOUNT`, `BITPOS`.

- **List**
    - Danh sách các chuỗi (string) được sắp xếp theo thứ tự chèn. Thực chất là một Linked List. 
    - Cấu trúc dữ liệu: Linked List (Quicklist).
    - Cho phép thêm/xóa phần tử ở đầu hoặc cuối danh sách với độ phức tạp O(1). Truy cập phần tử theo index có độ phức tạp O(N).
    - Sử dụng: Implementing queues (FIFO), stacks (LIFO), lưu trữ log gần dây, timeline feed.
    *   Lệnh chính: `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `LLEN`, `LRANGE`, `LINDEX`, `LSET`, `LTRIM`, `BLPOP`, `BRPOP` (blocking operations).

- **Set**
    - Tập hợp các chuỗi (string) duy nhất và không có thứ tự. 
    - Cấu trúc dữ liệu: Hash Table (Intset cho set nhỏ chỉ chứa số nguyên).
    - Cho phép thêm, xóa, lấy phần tử với độ phức tạp trung bình O(1).
    - Hỗ trợ các phép toán tập hợp: union (hợp), intersection (giao), difference (hiệu).
    - Sử dụng: Lưu trữ tags, theo dõi unique visitors, quản lý danh sách bạn bè/followers.
    *   Lệnh chính: `SADD`, `SREM`, `SMEMBERS`, `SISMEMBER`, `SCARD` (size), `SPOP` (remove random), `SRANDMEMBER` (get random), `SUNION`, `SINTER`, `SDIFF`.
    
- **Hash**
    - Lưu trữ dữ liệu dạng từ điển (key-value pairs) bên trong một key Redis chính. Tương tự như object trong lập trình. 
    - Cấu trúc dữ liệu: Hash Table (Ziplist cho hash nhỏ).
    *   Hiệu quả khi lưu trữ các object có nhiều trường (fields). Thay vì tạo nhiều key Redis riêng lẻ (vd: `user:1:name`, `user:1:email`), có thể lưu tất cả vào một hash `user:1`.
    *   Sử dụng: Lưu trữ thông tin profile người dùng, thông tin sản phẩm.
    *   Lệnh chính: `HSET`, `HGET`, `HDEL`, `HMSET`, `HMGET`, `HGETALL`, `HKEYS`, `HVALS`, `HLEN`, `HEXISTS`, `HINCRBY`.

*   **Sorted Set (ZSet):**
    *   Tập hợp các chuỗi (string) **duy nhất (unique)**, tương tự Set.
    * Cấu trúc dữ liệu: Skip List và Hash Table (Ziplist cho zset nhỏ).
    *   Điểm khác biệt chính: Mỗi phần tử (member) được liên kết với một **điểm số (score)** kiểu float. Các phần tử được sắp xếp theo score này. Nếu score bằng nhau, chúng được sắp xếp theo thứ tự từ điển.
    *   Cho phép truy cập, thêm, xóa phần tử nhanh chóng. Lấy phần tử theo rank (thứ hạng) hoặc theo khoảng score hiệu quả (độ phức tạp O(log N)).
    *   Sử dụng: Leaderboards, rate limiting, secondary indexing, autocomplete suggestions.
    *   Lệnh chính: `ZADD`, `ZREM`, `ZCARD`, `ZSCORE`, `ZCOUNT` (by score range), `ZRANK` (get rank by member), `ZREVRANK`, `ZRANGE` (by rank), `ZREVRANGE`, `ZRANGEBYSCORE`, `ZREVRANGEBYSCORE`.

* **Bitmap**
    - Không phải là một kiểu dữ liệu riêng biệt mà là một tập hợp các thao tác hoạt động trên kiểu String. Coi chuỗi như một mảng các bit. 
    - Cho phép thiết lập, xóa, và đếm các bit tại các vị trí (offset) cụ thể. Rất tiết kiệm bộ nhớ cho dữ liệu dạng boolean. 
    - Sử dụng: Theo dõi trạng thái on/off của người dùng (active/inactive), đếm số ngày active liên tục, phân tích cohort, access control. 
    *   Lệnh chính: `SETBIT`, `GETBIT`, `BITCOUNT` (đếm số bit 1), `BITPOS` (tìm vị trí bit 0 hoặc 1 đầu tiên), `BITOP` (AND, OR, XOR, NOT trên nhiều key).

*   **HyperLogLog (HLL):**
    *   Cấu trúc dữ liệu xác suất (probabilistic) dùng để **ước lượng** số lượng phần tử **duy nhất (cardinality)** trong một tập hợp rất lớn mà chỉ sử dụng một lượng bộ nhớ cố định và rất nhỏ (khoảng 12KB).
    *   Kết quả ước lượng có sai số chuẩn (standard error) khoảng 0.81%.
    *   Sử dụng: Đếm số lượng unique search queries, unique visitors trên website với lượng truy cập khổng lồ.
    *   Lệnh chính: `PFADD` (thêm phần tử), `PFCOUNT` (ước lượng cardinality), `PFMERGE` (gộp nhiều HLL).


*   **Geospatial:**
    *   Dựa trên Sorted Set, cho phép lưu trữ tọa độ địa lý (kinh độ, vĩ độ) của các địa điểm (members) và thực hiện các truy vấn dựa trên vị trí.
    *   Sử dụng: Tìm các địa điểm gần một điểm cho trước, tính khoảng cách giữa hai địa điểm.
    *   Lệnh chính: `GEOADD` (thêm địa điểm), `GEOPOS` (lấy tọa độ), `GEODIST` (tính khoảng cách), `GEORADIUS` / `GEORADIUSBYMEMBER` (tìm trong bán kính), `GEOHASH` (lấy geohash string).

*   **Streams:**
    *   Kiểu dữ liệu mới (từ Redis 5.0), là một cấu trúc dữ liệu **append-only log** (chỉ ghi thêm vào cuối).
    * Cấu trúc dữ liệu: RadixTree(giống B-tree) với macro node. 
    *   Mỗi entry trong stream có một ID duy nhất (thường dựa trên timestamp và sequence number) và một tập hợp các field-value pairs (tương tự Hash).
    *   Hỗ trợ **Consumer Groups**, cho phép nhiều client cùng đọc và xử lý các message trong stream một cách phối hợp, đảm bảo mỗi message chỉ được xử lý bởi một consumer trong group (hoặc có thể xử lý lại nếu cần).
    *   Sử dụng: Event sourcing, message queue bền vững hơn List, ghi log thời gian thực, giao tiếp giữa các microservices.
    *   Lệnh chính: `XADD` (thêm entry), `XRANGE`, `XREVRANGE` (đọc entries), `XLEN`, `XREAD` (đọc blocking), `XGROUP` (quản lý consumer groups), `XREADGROUP` (đọc bởi consumer group), `XACK` (xác nhận đã xử lý).

## 4. Redis Event Loop

Redis sử dụng mô hình **single-threaded event loop** để xử lý request, giúp đạt hiệu năng cao và tránh các vấn đề của đa luồng. 

**Cách Redis xử lý request với một luồng**

1. **Nhận dữ liệu từ client**: Redis server lắng nghe các kết nối đến trên một hoặc nhiều cổng mạng (sockets). Khi một client kết nối hoặc gửi dữ liệu, socket tương ứng trở nên "readable".
2. **I/O Multiplexing**: Redis sử dụng cơ chế I/O multiplexing của hệ điều hành (`epoll` trên Linux, `kqueue` trên MacOS). Thay vì tạo một luồng riêng cho mỗi client hoặc liên tục kiểm tra từng cổng socket (polling), Redis đăng ký tất cả các socket đang mở với kernel thông qua I/O multiplexing API. Redis chỉ cần gọi một hàm duy nhất (vd: `epoll_wait`) để đợi cho đến khi có bất kỳ socket nào sẵng sàng cho việc đọc (nếu có dữ liệu đến) hoặc ghi (buffer ghi trống). Hàm này sẽ block cho đến khi có sự kiện xảy ra hoặc timeout.
3. **Event Dispatcher**: Khi hàm I/O multiplexing trả về danh sách các socket có sự kiện, event loop sẽ duyệt qua danh sách này.
4. **File Event Handler (Đọc)**: Nếu sự kiện là "readable" (có dữ liệu từ client), Redis đọc dữ liệu từ socket vào buffer. Nó phân tích dữ liệu này để xác định lệnh Redis mà client yêu cầu. 
5. **Xử lý lệnh**: Lệnh được thực thi. Vì hầu hết lệnh Redis thao tác trên RAM và được tối ưu cao, chúng thực thi rất nhanh.
6. **File Event Handler (Ghi)**: Sau khi thực thi lệnh và có kết quả, Redis cần gửi kết quả về client. Nó ghi kết quả vào buffer ghi của socket tương ứng. Nếu buffer đầy, Redis sẽ đăng ký sự kiện "writable" cho socket đó với I/O multiplexing. Khi socket sẵn sàng để ghi (buffer trống), event loop sẽ được thông báo.
7. **Gửi kết quả về client**: Khi sự kiện "writable" xảy ra, Redis tiến hành ghi dữ liệu từ buffer của nó vào socket để gửi về client. 
8. **Time Event Handler**: Ngoài các sự kiện I/O (file events), event loop cũng xử lý các sự kiện hẹn giờ (time events), ví dụ như kiểm tra key hết hạn (expire), thực hiện snapshot RDB định kỳ, AOF rewrite, v.v.

**Cơ chế cốt lõi**
- **Socket**: Giao diện mạng chuẩn để giao tiếp client - server.
- **I/O Multiplexing**: Cho phép một luồng duy nhất giám sát nhiều kênh I/O (sockets) cùng lúc và chỉ xử lý những kênh đã sẵn sàng, tránh lãng phí CPU cho việc kiểm tra các kênh không hoạt động. Đây cũng là chìa khóa cho khả năng xử lý hàng nghìn kết nối đồng thời của Redis trên một luồng. 
- **Event-Driven Architecture**: Thay vì chờ đợi một tác vụ hoàn thành, Redis phản ứng với các sự kiện (dữ liệu đến, socket sẵn sàng ghi, timer hết hạn). Event loop là trung tâm điều phối các sự kiện này đến các trình xử lý (handlers) tương ứng.

Mô hình này là hiệu quả vì các thao tác Redis thường là non-blocking. Tuy nhiên cũng có rủi ro khi một lệnh chạy quá lâu (có thể do phức tạp hoặc sử dụng `pipeline`) có thể dẫn đến block event loop. 

## 5. Replication & Clustering (Sao chép và phân cụm)

Redis cung cấp các cơ chế để tăng cường tính sẵn sàng (high availability), khả năng đọc (read scalability) và khả năng ghi/lưu trữ (write/storage scalability).

**Replication (Master-Slave)**

- **Mục đích**: Sao chép dữ liệu từ một node Redis (Master) sang một hoặc nhiều node Redis khác (Slaves/Replicas).
- **Cách hoạt động**: 
    - Slave kết nối đến Master và gửi lệnh `PSYNC` (hoặc `SYNC` cho lần đầu).
    - Master bắt đầu quá trình BGSAVE để tạo snapshot RDB. Trong lúc này, nó buffer các lệnh ghi mới.
    - Master gửi file RDB cho Slave.
    - Slave xóa dữ liệu cũ (nếu có) và tải dữ liệu từ RDB. 
    - Master gửi các lệnh ghi đã buffer trong lúc tạo RDB cho slave.
    - Từ đó về sau, mỗi khi Master thực thi một lệnh ghi, nó sẽ gửi lệnh đó đến tất cả các Slave đang kết nối để chúng thực thi và duy trì trạng thái đồng bộ. Quá trình này là không đồng bộ (asynchronous).
- **Lợi ích**
    - **High Availability**: Nếu Master chết, một Slave có thể được thăng cấp (promoted) thành Master mới (quản lý bởi Redis Sentinel), giảm thiểu thời gian downtime.
    - **Read Scalability**: Các request đọc có thể được phân phối đến các Slave, giảm tải cho Master.
- Cấu hình: Sử dụng lệnh `REPLICAOF <master_ip> <master_port>` trên node Slave.

**Redis Cluster**

- **Mục đích**: Phân tán dữ liệu (sharding) và xử lý request trên nhiều node Redis, cho phép mở rộng cả dung lượng lưu trữ và khả năng ghi/đọc vượt quá giới hạn của một node đơn lẻ. 

- **Cách hoạt động**
    - **Data sharding**: Không gian key (keyspace) được chia thành **16,384 slot (hash slots)**.
    - Mỗi Master node trong cluster này chịu trách nghiệm quản lý một tập hợp các slot này.
    - Để xác định một key thuộc slot nào, Redis Cluster tính toán CRC16 của key và lấy modulo 16384: `slot = CRC16(key) % 16384`.
    - Khi client gửi lệnh đến một node, node đó sẽ tính toán slot của key. 
        - Nếu slot đó thuộc về node hiện tại, nó xử lý lệnh.
        - Nếu slot thuộc về một node khác, nó trả về một lỗi chuyển hướng (`MOVED error`) cho client, kèm theo địa chỉ của node đúng. Client sẽ cache lại map slot-node và tự động gửi request đến node đúng trong các lần sau. 

- **Ưu điểm**
    - **Scalability**: Cho phép mở rộng hệ thống Redis về dung lượng và thông lượng bằng cách thêm node. 
    - **High Availability**: Tích hợp sẵn cơ chế failover tự động. 

- **Nhược điểm**
    - Phức tạp hơn trong quản lý và vận hành so với setup đơn lẻ hoặc Master-Slave.
    - Các thao tác trên nhiều key (multi-key operations) chỉ được hỗ trợ nếu tất cả các key đó nằm cùng một slot (có thể dùng hash tags `{...}` để đảm bảo). Các thao tác cross-slot phức tạp hơn hoặc không hỗ trợ trực tiếp. 
    - Client cần hỗ trợ Redis Cluster protocol. 

**Redis Sentinel**
*   **Mục đích:** Cung cấp một giải pháp **high availability** cho kiến trúc **Redis Master-Slave** (không phải Redis Cluster). Sentinel là một hệ thống riêng biệt chạy song song với các node Redis Master/Slave.
*   **Chức năng chính:**
    *   **Monitoring (Giám sát):** Các tiến trình Sentinel liên tục kiểm tra tình trạng của các node Master và Slave.
    *   **Notification (Thông báo):** Thông báo cho quản trị viên hoặc các ứng dụng khác khi có sự cố xảy ra với các node Redis.
    *   **Automatic Failover (Tự động chuyển đổi dự phòng):** Nếu Sentinel xác định Master không còn hoạt động (down), nó sẽ khởi động quá trình failover:
        1.  Bầu chọn một Slave phù hợp nhất từ các Slave của Master đã chết.
        2.  Thăng cấp Slave được chọn thành Master mới.
        3.  Cấu hình các Slave còn lại để replicate từ Master mới.
        4.  Cập nhật cấu hình để các ứng dụng client biết địa chỉ của Master mới.
    *   **Configuration Provider (Cung cấp cấu hình):** Client kết nối đến Sentinel để hỏi địa chỉ của Master hiện tại, thay vì hardcode địa chỉ Master trong cấu hình client.
*   **Kiến trúc:** Sentinel thường được triển khai dưới dạng một cụm gồm ít nhất 3 tiến trình Sentinel chạy trên các máy chủ/VM khác nhau để tránh single point of failure cho chính hệ thống giám sát. Các Sentinel giao tiếp với nhau để đạt được sự đồng thuận (quorum) trước khi thực hiện failover.
*   **So sánh với Redis Cluster:** Sentinel tập trung vào high availability cho Master-Slave, không cung cấp data sharding. Redis Cluster cung cấp cả high availability và data sharding tích hợp sẵn.

## 6. Cơ chế tổ chức bộ nhớ trong Redis

Redis cực kỳ chú trọng đến việc sử dụng bộ nhớ hiệu quả.

*   **Đối tượng Redis (Redis Object):** Mọi key và value trong Redis được biểu diễn dưới dạng `redisObject`. Cấu trúc này chứa thông tin như:
    *   `type`: Kiểu dữ liệu (string, list, set, hash, zset).
    *   `encoding`: Cách biểu diễn dữ liệu cụ thể trong bộ nhớ (ví dụ: string có thể là `raw` hoặc `int`; list có thể là `ziplist` hoặc `linkedlist`). Redis chọn encoding tối ưu dựa trên kích thước và loại dữ liệu.
    *   `lru`: Thông tin cho thuật toán Least Recently Used (dùng cho eviction khi bộ nhớ đầy).
    *   `refcount`: Đếm số tham chiếu đến đối tượng (dùng cho quản lý bộ nhớ).
    *   `ptr`: Con trỏ đến dữ liệu thực tế.

*   **Tối ưu hóa Encoding:** Redis sử dụng các cấu trúc dữ liệu đặc biệt, tiết kiệm bộ nhớ cho các collection nhỏ:
    *   **`int` encoding:** Lưu số nguyên trực tiếp thay vì dạng chuỗi nếu giá trị nằm trong phạm vi nhất định.
    *   **`embstr` encoding:** Cho các chuỗi ngắn, `redisObject` và dữ liệu chuỗi được cấp phát trong một khối bộ nhớ liền kề, giảm phân mảnh và tăng hiệu quả cache.
    *   **`ziplist`:** Một cấu trúc dữ liệu được mã hóa đặc biệt, lưu trữ danh sách (List), hash (Hash), hoặc sorted set (ZSet) nhỏ một cách tuần tự trong một khối bộ nhớ liên tục. Nó tiết kiệm bộ nhớ hơn so với `linkedlist`, `hashtable`, hoặc `skiplist` nhưng có thể tốn CPU hơn cho các thao tác cập nhật/xóa ở giữa. Redis tự động chuyển đổi sang encoding hiệu quả hơn (`linkedlist`, `hashtable`, `skiplist`) khi collection vượt quá ngưỡng kích thước hoặc số lượng phần tử nhất định (cấu hình qua các tham số `*-max-ziplist-entries` và `*-max-ziplist-value`).
    *   **`intset`:** Dùng cho Set chỉ chứa các số nguyên. Lưu trữ các số nguyên đã sắp xếp trong một mảng liên tục, rất tiết kiệm bộ nhớ. Tự động chuyển sang `hashtable` nếu thêm phần tử không phải số nguyên hoặc set quá lớn.
    *   **`skiplist`:** Cấu trúc dữ liệu chính được sử dụng cho Sorted Set (cùng với một hash table để map member sang score). Cho phép tìm kiếm, thêm, xóa với độ phức tạp O(log N).

*   **Quản lý bộ nhớ (Memory Allocation):** Redis thường sử dụng các trình cấp phát bộ nhớ chuẩn như `malloc` (thường là `jemalloc` hoặc `tcmalloc` trên Linux, được tối ưu cho đa luồng và giảm phân mảnh) để quản lý heap.
*   **Memory Eviction:** Khi bộ nhớ đạt đến giới hạn `maxmemory` được cấu hình, Redis có thể loại bỏ (evict) các key để giải phóng không gian cho dữ liệu mới. Các chính sách eviction phổ biến bao gồm:
    *   `volatile-lru`: Xóa key ít được sử dụng gần đây nhất trong số các key có đặt `expire`.
    *   `allkeys-lru`: Xóa key ít được sử dụng gần đây nhất trong toàn bộ keyspace.
    *   `volatile-lfu`: Xóa key ít được sử dụng thường xuyên nhất (Least Frequently Used) trong số các key có `expire`.
    *   `allkeys-lfu`: Xóa key ít được sử dụng thường xuyên nhất trong toàn bộ keyspace.
    *   `volatile-random`: Xóa key ngẫu nhiên trong số các key có `expire`.
    *   `allkeys-random`: Xóa key ngẫu nhiên trong toàn bộ keyspace.
    *   `volatile-ttl`: Xóa key có thời gian sống (TTL) còn lại ngắn nhất.
    *   `noeviction`: Không xóa key, trả về lỗi khi bộ nhớ đầy.

Việc lựa chọn cấu trúc dữ liệu và encoding phù hợp, cùng với chính sách eviction hợp lý, là rất quan trọng để tối ưu hóa việc sử dụng bộ nhớ và hiệu năng của Redis.

## 7. Các thao tác Redis cơ bản

*   **General:**
    - `DEL key`: Xóa một key.  
    **Độ phức tạp**: O(N), với N là số lượng key.

    - `EXISTS key`: Kiểm tra sự tồn tại của key.  
    **Độ phức tạp**: O(1).

    - `EXPIRE key seconds`: Đặt thời gian hết hạn cho key (tính bằng giây).  
    **Độ phức tạp**: O(1).

    - `TTL key`: Lấy thời gian sống còn lại của key (tính bằng giây).  
    **Độ phức tạp**: O(1).

    - `FLUSHDB`: Xóa tất cả key trong database hiện tại.  
    **Độ phức tạp**: O(N).

*   **String:**
    - `SET key value`: Gán giá trị cho key.  
    **Độ phức tạp**: O(1).

    - `GET key`: Lấy giá trị của key.  
    **Độ phức tạp**: O(1).

    - `INCR key`: Tăng giá trị số nguyên của key lên 1.  
    **Độ phức tạp**: O(1).

    - `DECR key`: Giảm giá trị số nguyên của key đi 1.  
    **Độ phức tạp**: O(1).

*   **List:**
    - `LPUSH key element`: Thêm một phần tử vào đầu danh sách.  
    **Độ phức tạp**: O(1).

    - `RPUSH key element`: Thêm một phần tử vào cuối danh sách.  
    **Độ phức tạp**: O(1).

    - `LPOP key`: Xóa và trả về phần tử đầu tiên của danh sách.  
    **Độ phức tạp**: O(1).

    - `GET key`: Lấy phần tử ngẫu nhiên trong danh sách.
    **Độ phức tạp**: O(N). 

    - `RPOP key`: Xóa và trả về phần tử cuối cùng của danh sách.  
    **Độ phức tạp**: O(1).

    - `LLEN key`: Lấy độ dài (số phần tử) của danh sách.  
    **Độ phức tạp**: O(1).

*   **Set:**
    - `SADD key member`: Thêm một phần tử vào set.  
    **Độ phức tạp**: O(1).

    - `SREM key member`: Xóa một phần tử khỏi set.  
    **Độ phức tạp**: O(1).

    - `SMEMBERS key`: Lấy tất cả các phần tử trong set.  
    **Độ phức tạp**: O(N), với N là số phần tử trong set.

*   **Hash:**
    - `HSET key field value`: Gán giá trị cho một field trong hash.  
    **Độ phức tạp**: O(1).

    - `HGET key field`: Lấy giá trị của một field trong hash.  
    **Độ phức tạp**: O(1).

    - `HGETALL key`: Lấy tất cả các cặp field-value trong hash.  
    **Độ phức tạp**: O(N), với N là số lượng field.

*   **Sorted Set (ZSet):**
    - `ZADD key score member`: Thêm một phần tử với điểm số vào sorted set.  
    **Độ phức tạp**: O(log(N)).

    - `ZREM key member`: Xóa một phần tử khỏi sorted set.  
    **Độ phức tạp**: O(log(N)).

    - `ZRANGE key start stop`: Lấy các phần tử trong khoảng rank.  
    **Độ phức tạp**: O(log(N) + M), với M là số phần tử được trả về.

    *   `ZRANK key member`: Lấy thứ hạng (rank, bắt đầu từ 0) của phần tử (sắp xếp tăng dần theo score).
    **Độ phức tạp**: O(log(N)), với N là số phần tử trong sorted set.

## 9. Các mô hình caching phổ biến 

Khi sử dụng Redis làm cache, có nhiều chiến lược khác nhau để quản lý dữ liệu giữa ứng dụng, cache (Redis), và database chính.

### Cache-Aside (Lazy Loading)

Đây là mô hình phổ biến nhất, trong đó ứng dụng chịu trách nhiệm chính trong việc quản lý cache.

#### Cách hoạt động:
1. Ứng dụng kiểm tra cache (Redis) khi cần dữ liệu.
2. Nếu dữ liệu có trong cache (**cache hit**), ứng dụng sử dụng dữ liệu đó.
3. Nếu dữ liệu không có trong cache (**cache miss**), ứng dụng truy vấn từ database chính, sau đó lưu dữ liệu vào cache để sử dụng cho các lần truy cập sau.

#### Ưu điểm:
- Dễ triển khai và không phụ thuộc vào cache.
- Cache chỉ chứa dữ liệu được truy cập, giúp tiết kiệm bộ nhớ.

#### Nhược điểm:
- Có độ trễ cao hơn trong lần truy cập đầu tiên (**cache miss**).
- Dữ liệu trong cache có thể không đồng bộ với database nếu không có cơ chế làm mới hoặc xóa cache.

#### Trường hợp sử dụng:
- Các ứng dụng đọc dữ liệu thường xuyên nhưng ghi dữ liệu ít.
- Các hệ thống không yêu cầu dữ liệu trong cache phải luôn đồng bộ với database.

### Write-Through Cache

Đây là mô hình trong đó mọi thao tác ghi dữ liệu đều thực hiện cả vào cache lẫn database đồng thời. 

#### Cách hoạt động: 
1. Khi ứng dụng ghi dữ liệu mới (create/update), nó sẽ ghi vào cache trước. 
2. Sau đó, dữ liệu cũng được ghi xuống database ngay lập tức. 

#### Ưu điểm: 
- Cache và Database luôn đồng bộ.
- Tránh cache miss khi đọc dữ liệu ngay cả sau khi ghi.

#### Nhược điểm: 
- Độ trễ cao hơn vì phải ghi đồng thời 2 nơi.
- Nếu cache bị lỗi, có thể ảnh hưởng đến việc ghi dữ liệu. 

#### Trường hợp sử dụng: 
- Các ứng dụng yêu cầu đọc dữ liệu ngay sau khi ghi.
- Trường hợp dữ liệu thay đổi thường xuyên và cần đồng bộ tức thì.

### Write-Behind(Write-Back) Cache

Trong mô hình này, khi dữ liệu được ghi vào cache thì việc ghi vào database sẽ trì hoãn và thực hiện sau một khoảng thời gian, thường là theo batch.

#### Cách hoạt động: 
1. Ghi dữ liệu vào cache.
2. Một tiến trình nền sẽ ghi dữ liệu từ cache xuống database sau đó.

#### Ưu điểm: 
- Ghi nhanh hơn vì không cần chờ database.
- Tối ưu hiệu suất cho hệ thống ghi dữ liệu thường xuyên.

#### Nhược điểm: 
- Nguy cơ mất dữ liệu nếu hệ thống sập trước khi dữ liệu được ghi xuống database. 
- Độ phức tạp cao hơn, cần cơ chế đảm bảo đồng bộ. 

#### Trường hợp sử dụng: 
- Các hệ thống chịu tải ghi cao(như ghi log, sensor data).
- Không yêu cầu dữ liệu ghi phải phản ánh ngay lập tức trong database. 

### Read-Through Cache
Mô hình này tương tự với Cache-Aside nhưng ứng dụng không trực tiếp truy cập database khi cache miss. Thay vào đó, cache sẽ tự động truy vấn dữ liệu từ database nếu không có dữ liệu sẵn. 

#### Cách hoạt động:
1. Ứng dụng gọi `Get()` từ cache.
2. Nếu cache miss, cache layẻ tự động gọi xuống database, lấy dữ liệu, lưu lại và trả về kết quả cho ứng dụng.

#### Ưu điểm:
- Đơn giản hóa logic ứng dụng - chỉ cần gọi cache là đủ. 
- Quản lý cache tập trung, dễ kiểm soát TTL và refresh.

#### Nhược điểm: 
- Cần xây dựng lớp cache phức tạp hơn (cache phải biết truy cập database).
- Không linh hoạt nếu có nhiều nguồn dữ liệu.

#### Trường hợp sử dụng: 
- Khi muốn trừu tượng hóa việc đọc dữ liệu, để ứng dụng không cần lo kiểm tra cache/database.
- Hệ thống có logic đọc dữ liệu đơn giản và muốn tối ưu hóa kiến trúc. 


