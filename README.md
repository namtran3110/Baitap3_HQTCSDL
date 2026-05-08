# MÔN HỌC: HỆ QUẢN TRỊ CƠ SỞ DỮ LIỆU - ThS.ĐỖ DUY CỐP
## Họ và tên: Trần Nhất Nam
## MSSV: K235480106001
## Lớp: K59KMT

---

#### Nhiệm vụ 1: Thiết kế CSDL 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/bdf97ba4-2344-4707-b28c-7a878bb07f1d" />
- Khởi tạo Database với tên QuanLyThuNo_K235480106001

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/cec9c077-d256-40ea-ab00-40c7afa5eb19" />
- Khởi tạo các bảng chính của bài toán

```

-- Bảng Khách Hàng 
CREATE TABLE KhachHang (
    MaKhachHang INT PRIMARY KEY IDENTITY(1,1),
    HoTen NVARCHAR(100) NOT NULL,
    SoDienThoai VARCHAR(15),
    SoCanCuocCongDan VARCHAR(20) UNIQUE, 
    DiaChi NVARCHAR(255)
);

-- Bảng Hợp Đồng 
CREATE TABLE HopDong (
    MaHopDong INT PRIMARY KEY IDENTITY(1,1),
    MaKhachHang INT NOT NULL,
    NgayLapHopDong DATETIME DEFAULT GETDATE(),
    SoTienVayGoc DECIMAL(18, 2) NOT NULL, 
    HanChotMot DATE NOT NULL, -- Thay cho Deadline1
    HanChotHai DATE NOT NULL, -- Thay cho Deadline2
    TrangThaiHopDong NVARCHAR(50) DEFAULT N'Đang vay', 
    CONSTRAINT KhoaNgoai_HopDong_KhachHang FOREIGN KEY (MaKhachHang) REFERENCES KhachHang(MaKhachHang)
);

-- Bảng Tài Sản 
CREATE TABLE TaiSan (
    MaTaiSan INT PRIMARY KEY IDENTITY(1,1),
    MaHopDong INT NOT NULL,
    TenTaiSan NVARCHAR(100) NOT NULL,
    GiaTriDinhGia DECIMAL(18, 2) NOT NULL, 
    TrangThaiTaiSan NVARCHAR(50) DEFAULT N'Đang cầm cố', 
    CONSTRAINT KhoaNgoai_TaiSan_HopDong FOREIGN KEY (MaHopDong) REFERENCES HopDong(MaHopDong)
);

-- Bảng Nhật Ký Giao Dịch 
CREATE TABLE NhatKyGiaoDich (
    MaNhatKy INT PRIMARY KEY IDENTITY(1,1),
    MaHopDong INT NOT NULL,
    NgayGiaoDich DATETIME DEFAULT GETDATE(),
    SoTienTra DECIMAL(18, 2) NOT NULL, 
    NguoiThuTien NVARCHAR(50), 
    GhiChu NVARCHAR(255),
    CONSTRAINT KhoaNgoai_NhatKyGiaoDich_HopDong FOREIGN KEY (MaHopDong) REFERENCES HopDong(MaHopDong)
);
```

**Bảng các thực thể và thuộc tính(ERD)**

Dựa trên yêu cầu nghiệp vụ về quản lý cầm đồ, hệ thống được thiết kế với 4 thực thể chính đảm bảo chuẩn hóa dữ liệu 3NF.

| Thực thể | Thuộc tính | Diễn giải nghiệp vụ |
| :--- | :--- | :--- |
| **Khách Hàng** | `MaKhachHang`, `HoTen`, `SoDienThoai`, `SoCanCuocCongDan`, `DiaChi` | Lưu trữ thông tin định danh và liên lạc của người vay. |
| **Hợp Đồng** | `MaHopDong`, `MaKhachHang`, `NgayLapHopDong`, `SoTienVayGoc`, `HanChotMot`, `HanChotHai`, `TrangThaiHopDong` | Quản lý dòng tiền vay gốc, các mốc tính lãi và trạng thái thu hồi nợ. |
| **Tài Sản** | `MaTaiSan`, `MaHopDong`, `TenTaiSan`, `GiaTriDinhGia`, `TrangThaiTaiSan` | Quản lý chi tiết đồ thế chấp, phục vụ việc định giá và thanh lý khi quá `HanChotHai`. |
| **Nhật Ký Giao Dịch** | `MaNhatKy`, `MaHopDong`, `NgayGiaoDich`, `SoTienTra`, `NguoiThuTien`, `GhiChu` | Ghi lại lịch sử trả nợ từng phần, không làm mất dấu vết dòng tiền. |

**Mối quan hệ theo đúng yêu cầu của đề bài:**
* Một Khách Hàng có thể có nhiều Hợp Đồng.
* Một Hợp Đồng có thể thế chấp nhiều Tài Sản
* Một Hợp Đồng** có nhiều lần biến động tiền tệ được ghi lại trong Nhật Ký Giao Dịch

---

#### Nhiệm vụ 2: Cài đặt SQL







