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