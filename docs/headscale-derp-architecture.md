# Xây dựng VPN Overlay tự host với Headscale + DERP + WireGuard + Tailscale client

## 1. Tổng quan kiến trúc

Mô hình này chia làm 3 mặt phẳng rõ ràng:

-   Control plane: Headscale
    Quản lý identity, node registration, peer discovery, policy, route advertisement, DNS metadata. Headscale là bản self-host của coordination server kiểu Tailscale, không phải data relay mặc định.

-   Data plane: WireGuard trong các Tailscale client
    Sau khi node biết nhau, dữ liệu chạy trực tiếp giữa các peer bằng kết nối UDP mã hóa. Tailscale phân loại kiểu kết nối trực tiếp là direct connection qua UDP.

-   Relay plane: DERP
    DERP là đường dự phòng khi hai node không thể thiết lập direct peer-to-peer. DERP không phải đường đi mặc định cho mọi packet; nó là fallback relay và đồng thời hỗ trợ STUN để giúp client khám phá public endpoint phục vụ NAT traversal.

Kết luận bản chất
-   Headscale không chở data path.

-   WireGuard mới là đường dữ liệu chính.

-   DERP chỉ vào cuộc khi direct path fail.

-   Node không cần public static IP vì chúng không chờ inbound cố định như VPN kiểu cũ; chúng dùng peer discovery + NAT traversal +         outbound connectivity để tạo đường trực tiếp nếu môi trường NAT cho phép.

## 2. Kiến trúc mạng (Topology)

### Thành phần điển hình
-   VPS public
    -   Chạy:
        -   Headscale
        -   DERP self-host
        -   TLS endpoint
        -   metrics/logging nếu cần

-   Máy trong LAN / homelab
    -   server Proxmox
    -   VM lab
    -   DB node
    -   CI runner
    -   node subnet router

-   Laptop / client roaming
    -   máy cá nhân
    -   thiết bị di động
    -   node admin


### Luồng kết nối

#### Luồng 1: Control plane
Node khởi động Tailscale client, kết nối tới Headscale qua HTTPS/WebSocket để:
-   đăng ký
-   lấy peer list
-   lấy route/policy
-   lấy DERP map nếu có cấu hình custom DERP. Headscale yêu cầu public endpoint và reverse proxy phải hỗ trợ WebSocket; khi dùng embedded DERP còn cần mở UDP 3478 cho STUN.

#### Luồng 2: Direct peer-to-peer
Hai node thử thiết lập direct path:
-   xác định endpoint
-   gửi packet khám phá
-   punch NAT
-   nếu thành công, toàn bộ traffic đi thẳng node-to-node bằng UDP.

#### Luồng 3: Relay qua DERP
Nếu NAT/firewall quá gắt:
-   không punch được
    direct path thất bại
    client fallback qua DERP relay. DERP của Tailscale là relay over TLS/TCP và đáng tin cậy nhưng không tối ưu hiệu năng bằng direct UDP.
### ASCII topology

```text
                         Internet
                             |
                     +----------------+
                     | VPS Public     |
                     |----------------|
                     | Headscale      |
                     | DERP / STUN    |
                     | TLS endpoint   |
                     +--------+-------+
                              |
          -----------------------------------------------
          |                     |                      |
          | control plane       | control plane        | control plane
          |                     |                      |
+----------------+   direct?    |        direct?  +----------------+
| Laptop roaming |<------------>|<---------------->| Homelab node A |
| tailscale      |              |                  | tailscale       |
+----------------+              |                  +----------------+
          ^                     |                           ^
          |                     |                           |
          |                     |                           |
          |             +----------------+                  |
          +------------>| Homelab node B |<-----------------+
             direct?    | tailscale      |
                        +----------------+

Khi direct path fail:
Laptop / node A / node B  --->  DERP relay trên VPS  ---> peer còn lại
```

## 3. Cơ chế hoạt động (Deep dive)

### 3.1 Headscale quản lý identity, key, peer discovery thế nào

Headscale là coordination server tự host. Nó giữ metadata của tailnet kiểu self-managed:
-   user/namespace
-   node record
-   auth key / preauth key
-   tag
-   route advertisement
-   DNS info
-   policy ACL

Node có thể được đăng ký bằng preauth key. Quy trình cơ bản theo tài liệu Headscale là:
-   tạo user
-   tạo preauth key
-   client chạy tailscale up --login-server ... --authkey ...
-   node xuất hiện trong headscale nodes list.

Điểm cần nhớ: Headscale không encrypt/decrypt payload của peer traffic. Nó chỉ nói cho node biết:
-   ai đang trong mạng
-   public key của peer
-   endpoint nào nên thử
-   route/tag/policy nào được phép dùng

Đó là lý do tài nguyên VPS nhỏ vẫn sống khỏe khi đa số kết nối là direct.

### 3.2 WireGuard thiết lập tunnel ra sao

Tailscale client dùng WireGuard làm data plane. Về logic, mỗi peer cần:
-   public key của peer
-   identity mapping
-   dải IP overlay/AllowedIPs
-   endpoint hiện tại

Khi direct connection thành công, packet đi trực tiếp bằng UDP. Tailscale docs mô tả direct connection là thiết bị gửi packet trực tiếp cho nhau bằng UDP.

Điểm quan trọng:

-   Không có “VPN concentrator” kiểu cũ cho mọi luồng.
-   Không phải mọi gói đều qua VPS.
-   Overlay IP chỉ là lớp địa chỉ logic; đường thật có thể đổi theo endpoint NAT hiện tại.

### 3.3 NAT traversal và UDP hole punching

Node không có public static IP vẫn hoạt động được vì cơ chế chủ yếu là:
-   mỗi node kết nối outbound tới coordination/control path
-   node học public endpoint của nhau
-   cả hai bên thử gửi UDP tới endpoint học được
-   NAT giữ mapping đủ lâu để direct traffic hình thành

Headscale DERP docs nói DERP server còn dùng STUN trên UDP/3478 để giúp client khám phá public IP và NAT traversal.

Thực chiến:
-   NAT “dễ tính” → direct rất hay thành công
-   CGNAT/symmetric NAT/firewall enterprise → direct khó hơn, relay tăng

### 3.4 Vai trò thật sự của DERP

DERP không phải mặc định là middlebox cho mọi kết nối. Tailscale docs và blog đều mô tả DERP là fallback relay/path of last resort khi direct path không khả thi. Đồng thời coordination server phân phối DERP map để client biết relay nào khả dụng.

DERP có 2 giá trị:
-   reliability: mạng nào cho outbound HTTPS thì relay thường sống
-   last resort: giữ kết nối được ngay cả khi NAT traversal fail

DERP có 1 cái giá:
-   chậm hơn direct UDP
-   tốn tài nguyên VPS nếu relay traffic nhiều
-   thêm 1 hop mạng

### 3.5 Tại sao không phải VPN TCP truyền thống

Đường dữ liệu chuẩn của Tailscale/WireGuard là UDP direct. DERP fallback lại là TCP/TLS relay. Vì vậy:
-   trạng thái bình thường: VPN không phải TCP-centric
-   trạng thái xấu: có thể thấy traffic vòng qua relay over TLS/TCP

Tailscale mô tả direct connection là UDP và DERP là relay; blog NAT traversal của họ nêu DERP hiện là relay TCP-based over TLS, đáng tin cậy nhưng không tối ưu hiệu năng.

Nói ngắn:
-   đừng nhầm control/relay với data plane chính
-   nếu bạn thấy “mùi TCP”, nhiều khả năng bạn đang nhìn DERP hoặc control plane

## 4. Trade-off và giới hạn

### 4.1 Khi nào bắt buộc đi qua DERP

Các tình huống hay gặp:
-   symmetric NAT
-   CGNAT khó chịu
-   firewall chặn UDP
-   outbound policy bóp nghẹt hole punching
-   môi trường enterprise chỉ cho HTTPS ra ngoài

Khi đó direct path không hình thành, client fallback DERP.

### 4.2 Latency khi relay

Traffic relay qua DERP:
-   thêm 1 hop
-   chịu overhead TCP/TLS
-   không đẹp cho replication/chatty protocols/interactive SSH dài hạn

Tailscale nói DERP đáng tin cậy nhưng không tối ưu hiệu năng; TCP-based relay có thêm latency do handshake, buffering và đặc tính của TCP.

### 4.3 Bottleneck nếu VPS yếu

Nếu đa số traffic direct:
-   VPS chỉ làm coordination
-   tài nguyên thấp là đủ

Nếu nhiều traffic relay:
-   CPU tăng
-   bandwidth tăng
-   DERP thành choke point

Vì vậy, “Headscale nhẹ” là đúng; “DERP nhẹ” thì chỉ đúng nếu relay ít.

### 4.4 So sánh với các hướng khác
#### So với OpenVPN
-   OpenVPN thường thiên về mô hình concentrator/server rõ ràng hơn
-   thường TCP hoặc UDP nhưng vận hành nặng tay hơn
-   NAT traversal/P2P không phải thế mạnh cốt lõi kiểu Tailscale

#### So với WireGuard thuần
-   WireGuard raw rất gọn và nhanh
-   nhưng bạn phải tự xử:
    -   peer discovery
    -   dynamic endpoint
    -   auth flow
    -   route orchestration
    -   scale nhiều node
-   phù hợp static topology hơn
#### So với Tailscale cloud
-   vận hành nhàn hơn
-   control plane sẵn
-   DERP toàn cầu sẵn
-   đổi lại phụ thuộc bên thứ ba
Chốt trade-off
-   Headscale + DERP self-host: kiểm soát cao, học rất nhiều, nhưng bạn tự chịu vận hành
-   Tailscale official: ít đau đầu hơn
-   WireGuard raw: rất sạch, nhưng orchestration thủ công

## 5. Hướng dẫn triển khai thực tế

Phần này ưu tiên deploy tối thiểu chạy được, sau đó mới tối ưu.

### 5.1 Chuẩn bị VPS

Khuyến nghị:
-   Ubuntu 22.04+ hoặc Debian 12+ vì Headscale có gói DEB chính thức cho các bản này.
-   1 public IP
-   domain riêng, ví dụ vpn.example.com
-   mở port:
    -   80/tcp nếu dùng ACME HTTP-01
    -   443/tcp cho HTTPS và client access
    -   3478/udp cho STUN khi bật embedded DERP
    -   9090/tcp chỉ internal nếu muốn metrics/debug
    -   50443/tcp chỉ khi dùng gRPC remote control.
#### Tư duy triển khai
-   Đừng để Headscale chạy với user root nếu không cần.
-   Dùng systemd package chính thức nếu muốn đơn giản.
-   Reverse proxy chỉ thêm khi bạn thật sự cần chuẩn hóa TLS/ingress.
### 5.2 Cài Headscale

Theo docs, package DEB chính thức là đường đơn giản nhất trên Ubuntu/Debian và đã kèm systemd service, default config, local user.

Sau khi cài:

-   file config nằm ở /etc/headscale/config.yaml theo mặc định của Headscale YAML config path.

Kiểm tra config:

```text
headscale configtest
```

Lệnh này được tài liệu Headscale khuyến nghị để validate cấu hình.

### 5.3 Cấu hình Headscale server URL

Trong ``` config.yaml ```, mục bạn phải đúng ngay từ đầu là URL public mà client dùng để login. Tailscale docs có riêng mục “custom control server URL” và nêu rõ khi dùng Headscale thì client phải chỉ tới URL instance Headscale của bạn.

Ví dụ tư duy cấu hình:

-   URL phải là HTTPS public
-   cert hợp lệ
-   nếu reverse proxy thì phải support WebSocket
-   không đặt sau Cloudflare proxy/tunnel kiểu “ẩn origin” một cách ngây thơ; Headscale docs nói không hỗ trợ Cloudflare proxy/tunnel do yêu cầu WebSocket POST của giao thức Tailscale.
### 5.4 Cấu hình embedded DERP

Headscale docs có support DERP embedded và nhấn mạnh:
-   cần thêm port
-   cần STUN udp/3478
-   reverse proxy phải support WebSocket nếu dùng embedded DERP.

Tư duy cấu hình:
-   bật embedded DERP trong Headscale
-   gán region ID / region code / hostname rõ ràng
-   nếu muốn full private, đưa custom DERP map vào policy/derpMap để client ưu tiên DERP của bạn; Tailscale policy file hỗ trợ thêm custom DERP server và có thể disable DERP do Tailscale cung cấp nếu bạn có yêu cầu compliance/full control.
### 5.5 Tạo user / namespace

Headscale hiện dùng mô hình user. Theo docs:
```text
headscale users create homelab
```
rồi liệt kê user:
```text
headscale users list
```
Sau đó tạo preauth key. Quy trình này được tài liệu registration của Headscale mô tả.

### 5.6 Add node (client)

Trên node client đã cài Tailscale:
```text
tailscale up --login-server https://vpn.example.com --authkey <AUTH_KEY>
```
Tài liệu Tailscale về custom control server và Headscale registration đều chỉ ra logic này.

Sau đó trên server:
```text
headscale nodes list
```
để xác nhận node đã online.

### 5.7 Join mạng và kiểm tra

Lệnh thực dụng:
```text
tailscale status
tailscale ping <peer-name-or-ip>
ping <overlay-ip>
```
tailscale status cho bạn biết peer, route, trạng thái chung
tailscale ping giúp xác minh đường overlay
nếu ping không được, nhìn tiếp logs và xác định direct hay relay

Tailscale CLI docs có mô tả tailscale up, route, exit node, DNS flags; đây là CLI chính để vận hành client.

## 6. Cấu hình nâng cao

### 6.1 ACL / policy

Headscale có hỗ trợ ACLs. Đây là thứ biến overlay từ “VPN ai cũng thấy nhau” thành “Zero Trust nội bộ”.

Nguyên tắc:
-   deny by default
-   chỉ mở port/role cần thiết
-   tag node theo chức năng:
    -   tag:db
    -   tag:runner
    -   tag:admin
    -   tag:edge

Ví dụ mindset:
-   laptop admin được SSH vào tag:db
-   runner chỉ nói chuyện 443/8443 tới registry/internal API
-   db node không được nói lung tung ra toàn tailnet
### 6.2 Route subnet (subnet router)
Tailscale hỗ trợ subnet router để một node quảng bá route tới subnet vật lý phía sau nó. Docs phân biệt rất rõ:
-   subnet router: đưa tailnet chạm tới private subnet cụ thể
-   không phải route toàn bộ internet.
Dùng khi:
-   bạn chưa muốn cài Tailscale lên toàn bộ máy trong lab
-   bạn có 1 VM gateway đại diện cho cả subnet homelab

CLI liên quan:
```text
tailscale up --advertise-routes=192.168.50.0/24
```
Flag ```--advertise-routes``` được Tailscale CLI docs hỗ trợ.

### 6.3 Exit node

Exit node là route toàn bộ internet traffic qua một node khác; docs Tailscale mô tả nó như một VPN server cho tailnet.

CLI liên quan:
```text
tailscale up --advertise-exit-node
tailscale set --exit-node=<name-or-ip>
```
```--exit-node``` và ```--exit-node-allow-lan-access``` đều có trong CLI docs.

Use-case:
-   laptop đi công tác muốn egress qua VPS/home lab
-   cần geo-egress ổn định
-   cần tập trung log outbound
### 6.4 DNS nội bộ

Tailscale hỗ trợ MagicDNS và DNS settings; tài liệu nêu MagicDNS giúp tự gán tên DNS cho thiết bị trong tailnet và DNS có thể được quản lý qua setting/CLI.

Khuyến nghị:
-   bật MagicDNS cho lab nhỏ
-   dùng split DNS nếu bạn có domain nội bộ thật
-   tách tên:
    -   db-01.tail
    -   runner-01.tail
    -   pve-01.tail
### 6.5 Multi-node scale

Khi scale từ 3 node lên vài chục node:
-   bỏ tư duy “thêm peer bằng tay”
-   dùng tag và auth key ngắn hạn
-   tách role bằng ACL
-   có naming convention từ đầu
-   chọn node nào được advertise route/exit node rõ ràng

Headscale feature docs liệt kê support cho tags, routes, subnet routers, exit nodes, MagicDNS, split DNS.

## 7. Bảo mật

### 7.1 Key rotation

Node identity/key sống lâu quá là mùi rủi ro.
Thực chiến:
-   dùng preauth key ngắn hạn cho bootstrap
-   không nhét auth key cố định vào dotfile mãi mãi
-   revoke node cũ khi máy bị bỏ/đổi owner

Tài liệu Headscale registration cho biết preauth key mặc định có thể dùng 1 lần và chỉ hợp lệ trong thời gian giới hạn nếu bạn dùng default setting.

### 7.2 Access control

Bảo mật của mô hình này không nằm ở “có mã hóa”.
Mã hóa chỉ là baseline.

Phần ăn tiền là:
-   identity-based access
-   ACL đúng vai trò
-   tách route subnet
-   không cấp exit node bừa bãi
### 7.3 Rủi ro khi self-host

Bạn tự host thì tự gánh:
-   mất cert/TLS
-   reverse proxy sai → client chết
-   DERP map lỗi → fallback hỏng
-   policy quá rộng → biến tailnet thành LAN phẳng
-   update lệch version → khó debug

Thêm nữa, docs Headscale nói bản thân họ không dùng reverse proxy trong môi trường chính của họ; reverse proxy thường là chỗ gây lỗi cộng đồng hay gặp.

### 7.4 Hardening VPS

Checklist tối thiểu:
-   chỉ mở đúng port cần thiết
-   SSH key only
-   fail2ban nếu có SSH public
-   patch định kỳ
-   tách user chạy service
-   không nhét mọi thứ vào cùng một VPS nếu không cần
-   logrotate
-   backup config + key material + DB metadata
-   firewall host: chỉ cho inbound 443/tcp, 3478/udp, và port quản trị thật sự cần

## 8. Use-case thực tế cho DevSecOps

### 8.1 Remote homelab

Bạn ở ngoài đường vẫn SSH, RDP, truy cập dashboard Proxmox, Grafana, Git service, registry nội bộ mà không cần expose từng cổng ra Internet.

### 8.2 CI/CD runner private network

Runner nằm trong tailnet, build/deploy vào:
-   private registry
-   internal staging
-   DB migration endpoint
-   Kubernetes API private
-   Không cần mở inbound public chỉ để runner nói chuyện.
### 8.3 Kết nối multi-site lab

Home lab + VPS + laptop + máy công ty + mini site khác có thể vào cùng overlay.
Khi direct được thì nhanh.
Khi direct không được thì DERP cứu.

### 8.4 Zero Trust nội bộ

Thay vì “vào VPN là thấy hết”, bạn áp policy:

-   dev chỉ thấy dev
-   db admin mới chạm DB
-   runner chỉ chạm deploy target

Đây mới là tư duy DevSecOps đúng bài.

## 9. Best practices

### 9.1 Khi nào nên dùng DERP riêng

Nên tự host DERP khi:

-   bạn muốn full control
-   muốn giảm phụ thuộc vào relay bên thứ ba
-   có node ở các mạng rất hostile
-   compliance không cho relay qua hạ tầng người khác
-   muốn đo và tối ưu relay path của riêng mình
### 9.2 Khi nào không cần DERP riêng

Không cần vội nếu:
-   tailnet nhỏ
-   node chủ yếu direct được
-   bạn chỉ lab vài máy
-   chưa có nhu cầu compliance
-   không muốn tự gánh thêm 1 lớp vận hành
### 9.3 Tối ưu hiệu năng
-   đặt VPS gần đa số client
-   ưu tiên direct path, xem DERP là fallback
-   nếu một site luôn bị relay, kiểm tra NAT/firewall trước khi nâng VPS
-   đừng để ứng dụng nặng đi qua exit node/relay vô tội vạ
-   với DB replication/chatty traffic, direct path gần như là bắt buộc nếu muốn mượt
### 9.4 Monitoring

Ít nhất nên theo dõi:
-   node online/offline
-   route được advertise
-   tỷ lệ direct vs relay
-   latency ping overlay
-   CPU/RAM/BW trên VPS
-   log Headscale
-   cert expiry

Headscale docs nêu có metrics/debug endpoint trên 9090/tcp và không nên public nó.

## 10. Kết luận

Khi nào nên chọn mô hình này

Chọn Headscale + self-host DERP + Tailscale client khi bạn muốn:
-   private control plane
-   hiểu sâu networking thật sự
-   overlay network sạch, linh hoạt
-   zero trust nội bộ
-   homelab có chiều sâu kỹ thuật
-   tự làm lab/infra để học DevSecOps nghiêm túc

Đây là lựa chọn đẹp khi bạn cần:
-   ít phụ thuộc bên thứ ba
-   node không có public static IP
-   vẫn muốn roaming + direct P2P + fallback relay
-   Khi nào nên dùng Tailscale official cloud

Dùng cloud khi:
-   bạn ưu tiên tốc độ triển khai hơn tự chủ
-   bạn không muốn vận hành cert/reverse proxy/DERP
-   bạn không muốn gánh incident control plane
-   tailnet nhỏ và không có yêu cầu compliance đặc biệt
Chốt thẳng

Mô hình này không “thần kỳ”. Nó chỉ làm rất tốt 3 việc:
-   tách control plane khỏi data plane
-   ưu tiên direct WireGuard UDP
-   có DERP cứu khi NAT traversal fail

Với homelab hoặc hạ tầng cá nhân nghiêm túc, đây là một trong những cách đẹp nhất để tiến từ “VPN cho có” sang “private overlay network có tư duy hệ thống”.

Checklist triển khai ngắn
-   Có VPS public, domain, TLS
-   Cài Headscale bằng package chính thức
-   Bật embedded DERP nếu cần full control
-   Mở 443/tcp và 3478/udp
-   Tạo user + preauth key
-   Client tailscale up --login-server ... --authkey ...
-   Kiểm tra tailscale status, tailscale ping
-   Viết ACL sớm, đừng để tailnet phẳng
-   Chỉ dùng relay như fallback, không coi đó là đường mặc định