# Hạ tầng mạng doanh nghiệp đa chi nhánh trên VMware

> Đồ án học phần **Quản trị mạng** – Trường Đại học Công Thương TP.HCM (HUIT), Khoa Công nghệ Thông tin
> GVHD: Ths. Nguyễn Quốc Sử · Mã lớp: 010110216903 · **Nhóm 7** · Tháng 06/2026

## Giới thiệu

Dự án xây dựng một mô hình mạng doanh nghiệp mô phỏng trên **VMware Workstation**, gồm 3 khu vực hoạt động là **HQ (trụ sở chính)**, **KHO (Warehouse)** và **SẢN XUẤT (Production)**. Hệ thống được thiết kế theo hướng phân tầng, phân vùng mạng, quản trị tập trung, kết nối liên khu vực, hỗ trợ làm việc từ xa qua VPN và có giám sát vận hành.

Mô hình không chỉ "chạy được" mà hướng tới các giá trị của một hạ tầng doanh nghiệp thực tế:

- **Ổn định:** hệ thống chia thành nhiều vùng độc lập, một vùng lỗi không làm tê liệt toàn công ty.
- **Bảo mật:** dữ liệu nhạy cảm (Kế toán, Điều hành) nằm ở vùng mạng riêng; dịch vụ public (Web, VPN) tách khỏi mạng nội bộ.
- **Dễ quản lý, dễ mở rộng:** thêm nhân sự, phòng ban hoặc chi nhánh mà không phải thiết kế lại; có thể cô lập nhanh thiết bị bất thường.
- **Làm việc từ xa:** nhân viên kết nối vào mạng nội bộ qua VPN.

## Sơ đồ hạ tầng

![Sơ đồ logic hạ tầng mạng](diagrams/so-do-logic-ha-tang-mang.png)

## Những gì đã triển khai

### 1. Thiết kế mạng và quy hoạch IP
- Chia mạng thành các vùng riêng cho từng chức năng: Server, Office, Sensitive, Web/VPN, Warehouse, Production và Transit liên khu vực.
- Quy hoạch IP theo từng phòng ban, có dải dự phòng để mở rộng.
- Thiết kế DHCP Scope cho từng mạng người dùng.
- Mô phỏng WAN giữa các khu vực bằng mạng Transit (VMnet9) và router MikroTik.

| Mạng | VMnet | Subnet | Gateway |
|---|---|---|---|
| HQ – Web/VPN | VMnet2 | 10.10.200.0/27 | 10.10.200.1 |
| HQ – Server | VMnet3 | 10.10.10.0/24 | 10.10.10.1 |
| HQ – Office | VMnet4 | 10.10.20.0/24 | 10.10.20.1 |
| HQ – Sensitive | VMnet5 | 10.10.30.0/25 | 10.10.30.1 |
| KHO | VMnet6 | 10.20.10.0/24 | 10.20.10.1 |
| SẢN XUẤT | VMnet7 | 10.30.10.0/24 | 10.30.10.1 |
| Transit (WAN giả lập) | VMnet9 | 172.16.0.0/29 | HQ .1 · KHO .2 · SX .3 |
| VPN Users | – | 10.99.99.0/27 | 10.99.99.1 |

### 2. Ba khu vực hoạt động

| Khu vực | Phòng ban | Thành phần chính |
|---|---|---|
| **HQ** | Điều hành, Kế toán, HR, IT, Kinh doanh, Nhân viên văn phòng | Toàn bộ dịch vụ lõi: AD DS, DNS, DHCP, File Server, Monitoring, Web + VPN |
| **KHO** | Quản lý, Logistics/Kiểm kê, Vận hành kho | Router MikroTik riêng, RODC-WH |
| **SẢN XUẤT** | R&D, Quản lý, Vận hành sản xuất | Router MikroTik riêng, RODC-PROD |

Ngoài ra có nhóm nhân sự làm việc từ xa (Sales, nhân viên công tác) truy cập qua VPN.

### 3. Máy chủ và vai trò

| Máy chủ | IP | Vai trò |
|---|---|---|
| SRV-DC1 | 10.10.10.10 | Active Directory Domain Services, DNS, DHCP |
| SRV-FILE1 | 10.10.10.20 | File Server, FSRM, DFS Namespace, Backup/Restore |
| SRV-MON1 | 10.10.10.30 | Giám sát PRTG |
| WEB1 / VPN1 | 10.10.200.10 / .20 | Web Server, VPN Remote Access |
| RODC-WH | 10.20.10.10 | Read-Only Domain Controller khu KHO |
| RODC-PROD | 10.30.10.10 | Read-Only Domain Controller khu SẢN XUẤT |

### 4. Active Directory
- Domain **abc.local**, Forest Functional Level Windows Server 2022.
- Tổ chức OU theo mô hình *Khu vực → Phòng ban → Users / Computers / Groups* (HQ, KHO, SANXUAT, Servers).
- Nhóm người dùng theo phòng ban (`GRP_IT`, `GRP_HR`, `GRP_ACCOUNTING`, …), nhóm quản trị tài nguyên (`GRP_FS_FULL`, `GRP_FS_RW`, `GRP_FS_RO`, `GRP_DFS_ADMIN`) và nhóm VPN/Remote.
- Quy tắc đặt tên tài khoản `firstname.lastname`.
- Chính sách mật khẩu: lịch sử 5, tối đa 90 ngày, tối thiểu 1 ngày, độ dài tối thiểu 8, bật độ phức tạp.
- Join domain cho máy trạm và máy chủ.
- Triển khai RODC tại hai chi nhánh để xác thực cục bộ và giảm phụ thuộc đường truyền về HQ.

### 5. File Server và quản trị tài nguyên
- Chia sẻ dữ liệu chung và dữ liệu theo phòng ban, phân quyền bằng **Share Permission + NTFS Permission** theo nhóm (trưởng phòng Modify, nhân viên Read).
- **Drive Mapping** bằng GPO cho từng phòng ban.
- **DFS Namespace** (domain-based) để truy cập dữ liệu tập trung.
- **FSRM:** đặt Quota dung lượng và File Screening chặn loại tệp không cho phép.
- **Backup/Restore** bằng Windows Server Backup, có kiểm thử khôi phục dữ liệu.

### 6. Chính sách nhóm (GPO)
- Password Policy, Screen Lock, DFS Access, Drive Mapping cho từng khu vực.
- Chặn USB (Removable Storage) và hạn chế Control Panel cho phòng Kế toán.
- Quota Policy cho khu KHO; RODC Authentication cho khu SẢN XUẤT.

### 7. VPN truy cập từ xa
- Dịch vụ Routing and Remote Access (RRAS), cấu hình VPN access + LAN routing.
- Xác thực bằng Windows Authentication, cấp quyền Dial-in qua Active Directory.
- Kiểm thử kết nối từ máy client và theo dõi phiên kết nối trên server.

### 8. Giám sát hệ thống
- **PRTG Network Monitor** đặt tại SRV-MON1.
- Giám sát Ping, CPU, Disk, Memory, Uptime cho các máy chủ (SRV-FILE1, SRV-MON1, WEB1, …).
- Thêm Device/Sensor thủ công và kiểm tra trạng thái trên dashboard.

## Khả năng duy trì khi có sự cố

| Sự cố | Ảnh hưởng | Hệ thống còn hoạt động |
|---|---|---|
| SRV-DC1 | Khó xác thực người dùng mới, DNS/DHCP trung tâm | RODC hỗ trợ xác thực chi nhánh; phiên đã đăng nhập vẫn dùng được ngắn hạn |
| SRV-FILE1 | Mất truy cập thư mục chia sẻ, DFS, FSRM | ADDS, DNS, DHCP và đăng nhập domain bình thường |
| SRV-MON1 | Mất giám sát và cảnh báo | Toàn bộ mạng và domain vẫn chạy |
| Web + VPN | Không truy cập web, không VPN từ xa | Mạng nội bộ, ADDS và File Server không ảnh hưởng |
| RODC chi nhánh | Chi nhánh xác thực về HQ, đăng nhập chậm hơn | Hoạt động nếu đường Transit còn ổn định |

## Điểm mạnh và hạn chế

**Điểm mạnh:** phân tách vùng mạng rõ ràng; tách dịch vụ public khỏi mạng nội bộ; có vùng Sensitive riêng; có RODC cho chi nhánh; có Monitoring riêng; DHCP quản trị tập trung; có mô hình WAN giả lập nhiều site.

**Hạn chế:** mỗi thành phần chỉ có một bản (1 DC, 1 File Server, 1 Monitoring, 1 VPN, 1 Core Router HQ) nên chưa có dự phòng; nếu HQ gặp sự cố thì ADDS, DNS, DHCP, File, VPN và định tuyến WAN đều gián đoạn.

## Công nghệ sử dụng

VMware Workstation (VMnet host-only) · Windows Server 2022 · Active Directory Domain Services · DNS · DHCP · File Server Resource Manager · Distributed File System · Group Policy · Windows Server Backup · Routing and Remote Access · MikroTik RouterOS · PRTG Network Monitor

## Phạm vi

Đây là mô hình lab phục vụ học tập, tập trung mô phỏng logic hoạt động của hạ tầng doanh nghiệp. Dự án **không** bao gồm High Availability Cluster, SAN Storage, Load Balancer, Firewall HA và WiFi vật lý. Các Access Point (nếu có) chỉ mang tính biểu diễn trên sơ đồ.

## Cấu trúc repository

```
.
├── README.md
├── diagrams/
│   └── so-do-logic-ha-tang-mang.png    # Sơ đồ logic hạ tầng
└── docs/
    └── Nhom7_BaoCao.pdf                # Báo cáo đồ án đầy đủ
```

## Thành viên – Nhóm 7

| Thành viên | Phụ trách |
|---|---|
| Đỗ Nguyễn Quốc Huy | Thiết kế sơ đồ logic, chia subnet/VLAN, quy hoạch IP, bảng tài nguyên; sơ đồ luồng dịch vụ và kịch bản mở rộng/thu hẹp |
| Nguyễn Hồng Linh | File Server, NTFS/Share Permission, Drive Mapping, FSRM, GPO, Backup/Restore; chuẩn hóa báo cáo; hỗ trợ quản lý nhóm |
| Nguyễn Thị Minh Thư | RODC, chính sách máy trạm, VPN client-to-site; thu thập minh chứng, video demo, nhật ký nhóm |
| Võ Văn Thái | Cài Windows Server, AD DS, DNS, DHCP, join domain, OU/user/group, account policy; hỗ trợ triển khai PRTG |

---
*Dự án phục vụ mục đích học tập. GVHD: Ths. Nguyễn Quốc Sử.*
