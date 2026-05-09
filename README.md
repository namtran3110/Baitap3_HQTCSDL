# MÔN HỌC: HỆ QUẢN TRỊ CƠ SỞ DỮ LIỆU - ThS.ĐỖ DUY CỐP
## Họ và tên: Trần Nhất Nam
## MSSV: K235480106001
## Lớp: K59KMT

---

### Nhiệm vụ 1: Thiết kế CSDL 
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

### Nhiệm vụ 2: Cài đặt SQL
### ENVENT 1: ĐĂNG KÝ HỢP ĐỒNG MỚI
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f3433973-7fbe-4060-8de5-532a0321e3ab" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/81b6981f-51b0-4f14-8eb2-d71622aaf822" />
- Store Procedure tiếp nhận hợp đồng vay của khách hàng

```
USE QuanLyThuNo_K235480106001;
GO

CREATE PROCEDURE SpDangKyHopDongMoi
    -- Thông tin khách hàng
    @HoTen NVARCHAR(100),
    @SoCanCuocCongDan VARCHAR(20),
    @SoDienThoai VARCHAR(15),
    @DiaChi NVARCHAR(255),
    
    -- Thông tin hợp đồng
    @SoTienVayGoc DECIMAL(18, 2),
    @HanChotMot DATE,
    @HanChotHai DATE,
    
    -- Danh sách tài sản (Truyền vào dưới dạng chuỗi JSON)
    @DanhSachTaiSanJson NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Bắt đầu một giao dịch (Đảm bảo tính toàn vẹn dữ liệu)
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- ==============================================================
        -- BƯỚC 1: XỬ LÝ THÔNG TIN KHÁCH HÀNG
        -- ==============================================================
        DECLARE @MaKhachHangHienTai INT;
        
        -- Kiểm tra xem khách hàng đã tồn tại trong hệ thống chưa (dựa vào CCCD)
        SELECT @MaKhachHangHienTai = MaKhachHang 
        FROM KhachHang 
        WHERE SoCanCuocCongDan = @SoCanCuocCongDan;
        
        -- Nếu chưa tồn tại -> Thêm khách hàng mới
        IF @MaKhachHangHienTai IS NULL
        BEGIN
            INSERT INTO KhachHang (HoTen, SoCanCuocCongDan, SoDienThoai, DiaChi)
            VALUES (@HoTen, @SoCanCuocCongDan, @SoDienThoai, @DiaChi);
            
            -- Lấy ID khách hàng vừa được sinh ra
            SET @MaKhachHangHienTai = SCOPE_IDENTITY();
        END
        
        -- ==============================================================
        -- BƯỚC 2: TẠO HỢP ĐỒNG VAY TIỀN
        -- ==============================================================
        DECLARE @MaHopDongMoi INT;
        
        INSERT INTO HopDong (MaKhachHang, NgayLapHopDong, SoTienVayGoc, HanChotMot, HanChotHai, TrangThaiHopDong)
        VALUES (@MaKhachHangHienTai, GETDATE(), @SoTienVayGoc, @HanChotMot, @HanChotHai, N'Đang vay');
        
        -- Lấy ID hợp đồng vừa sinh ra
        SET @MaHopDongMoi = SCOPE_IDENTITY();
        
        -- ==============================================================
        -- BƯỚC 3: XỬ LÝ DANH SÁCH TÀI SẢN (Phân tích cú pháp JSON)
        -- ==============================================================
        -- Sử dụng OPENJSON để đọc mảng tài sản truyền vào và insert vào bảng
        INSERT INTO TaiSan (MaHopDong, TenTaiSan, GiaTriDinhGia, TrangThaiTaiSan)
        SELECT 
            @MaHopDongMoi,
            JSON_VALUE(value, '$.TenTaiSan'),
            CAST(JSON_VALUE(value, '$.GiaTriDinhGia') AS DECIMAL(18,2)),
            N'Đang cầm cố'
        FROM OPENJSON(@DanhSachTaiSanJson);
        
        -- Nếu tất cả các bước đều thành công thì lưu dữ liệu vật lý
        COMMIT TRANSACTION;
        PRINT N'>> Tạo hợp đồng và lưu danh sách tài sản thành công!';
        
    END TRY
    BEGIN CATCH
        -- Nếu có bất kỳ lỗi nào xảy ra ở 3 bước trên, quay lui (Rollback) toàn bộ
        IF @@TRANCOUNT > 0 
            ROLLBACK TRANSACTION;
            
        -- Hiển thị thông báo lỗi hệ thống
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
    END CATCH
END;
GO
```

**Cơ chế đăng ký hợp đồng**
Code thực thi theo 3 cơ chế kỹ thuật cốt lõi:
•	Cơ chế chống rác dữ liệu: Toàn bộ quá trình "tạo Khách hàng -> Tạo Hợp đồng -> Thêm Tài sản" được bọc trong một "Giao dịch" (BEGIN TRAN). Nếu mất điện hoặc lỗi ở bước thêm tài sản, hệ thống sẽ lập tức quay xe (ROLLBACK), hủy bỏ toàn bộ, đảm bảo không bao giờ có chuyện "Có hợp đồng mà không có tài sản".
•	Cơ chế tái sử dụng Khách hàng: Hệ thống tự động dùng SoCanCuocCongDan để tra cứu. Nếu là khách cũ, nó sẽ nối hợp đồng mới vào mã khách cũ. Nếu là khách mới, nó tự tạo và lấy ngay mã vừa sinh ra bằng lệnh SCOPE_IDENTITY().
•	Cơ chế xử lý mảng bằng JSON: Thay vì phải viết vòng lặp lằng nhằng hoặc gọi hàm nhiều lần để lưu nhiều tài sản, code dùng OPENJSON để "mổ xẻ" một chuỗi JSON và đẩy toàn bộ danh sách tài sản vào Database chỉ bằng 1 câu lệnh INSERT duy nhất.

**Cơ chế hoạt động của Deadline 1 và 2 (HanChotMot, HanChotHai)**
Hai mốc thời gian này được người dùng tự ấn định khi lập hợp đồng và đóng vai trò như "Hai chiếc công tắc tự động" của hệ thống:
•	HanChotMot (Công tắc chuyển đổi Lãi suất & Nợ xấu): Cơ chế tính tiền (ở Event 2) sẽ luôn lấy "Ngày hiện tại" để so sánh với mốc này.
o	Nếu Ngày hiện tại $\le$ HanChotMot: Hệ thống tính Lãi đơn (5000đ/1tr/ngày) êm đềm.
o	Nếu Ngày hiện tại > HanChotMot: Công tắc bật! Hợp đồng tự động bị liệt vào danh sách đen "Quá hạn", đồng thời thuật toán chuyển sang tính Lãi kép (Lãi mẹ đẻ lãi con) cực kỳ gắt gao.
•	HanChotHai (Công tắc Thanh lý tài sản):
Đây là giới hạn chịu đựng cuối cùng của tiệm cầm đồ. Khi ngày hiện tại vượt qua mốc này mà khách vẫn chưa trả hết tiền, "công tắc 2" kích hoạt. Hệ thống sẽ tự động gỡ mác "Đang cầm cố" của các món đồ (Laptop, Xe máy...) và đóng dấu "Sẵn sàng thanh lý" để chủ tiệm có quyền mang ra chợ bán bù vốn.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/8278847b-c027-465f-a8be-8604c959ab88" />
- Chèn thử 1 khách hàng "Nguyễn Văn A", kết quả cho ra là bảng bên dưới:


**Bảng KhachHang (Khách Hàng):**
| MaKhachHang | HoTen | SoDienThoai | SoCanCuocCongDan | DiaChi |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Nguyễn Văn A | 0901234567 | 001200300400 | Hà Nội |

**Bảng HopDong (Hợp Đồng):**
| MaHopDong | MaKhachHang | NgayLapHopDong | SoTienVayGoc | HanChotMot | HanChotHai | TrangThaiHopDong |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | 1 | 2026-05-09 | 10000000.00 | 2026-06-01 | 2026-07-01 | Đang vay |

**Bảng TaiSan (Tài Sản Thế Chấp):**
| MaTaiSan | MaHopDong | TenTaiSan | GiaTriDinhGia | TrangThaiTaiSan |
| :--- | :--- | :--- | :--- | :--- |
| 1 | 1 | Laptop Dell XPS 15 | 12000000.00 | Đang cầm cố |
| 2 | 1 | iPhone 14 Pro Max | 8000000.00 | Đang cầm cố |

---
#### Event 2: Tính toán công nợ thời gian thực 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/aacb8533-ed58-4fda-8bb4-4ae2b4fcc97b" />
- Hàm 1: Tính toán công nợ của 1 Hợp Đồng

Hàm này sẽ nhận vào Mã hợp đồng và Ngày muốn tra cứu. Thuật toán tự động xét xem đã vượt HanChotMot chưa để áp dụng Lãi đơn hay Lãi kép. Đồng thời ưu tiên trừ số tiền khách đã trả góp vào phần Lãi trước, dư mới trừ Gốc.

```
USE QuanLyThuNo_K235480106001;
GO

IF OBJECT_ID('FnTinhTienHopDong', 'FN') IS NOT NULL DROP FUNCTION FnTinhTienHopDong;
GO

CREATE FUNCTION FnTinhTienHopDong (
    @MaHopDong INT,
    @NgayTinhToan DATE
)
RETURNS DECIMAL(18,2)
AS
BEGIN
    DECLARE @SoTienVayGoc DECIMAL(18,2);
    DECLARE @NgayLapHopDong DATE;
    DECLARE @HanChotMot DATE;
    
    -- Lấy thông tin gốc của hợp đồng
    SELECT 
        @SoTienVayGoc = SoTienVayGoc, 
        @NgayLapHopDong = CAST(NgayLapHopDong AS DATE), 
        @HanChotMot = HanChotMot
    FROM HopDong 
    WHERE MaHopDong = @MaHopDong;

    -- ==========================================
    -- BƯỚC 1: TÍNH TỔNG TIỀN LÃI (Chưa trừ trả góp)
    -- Lãi suất 5000đ/1 triệu/ngày = 0.5% = 0.005
    -- ==========================================
    DECLARE @TongLai DECIMAL(18,2) = 0;
    DECLARE @LaiSuatNgay FLOAT = 0.005; 
    
    IF @NgayTinhToan <= @HanChotMot
    BEGIN
        -- ÁP DỤNG LÃI ĐƠN
        DECLARE @SoNgay INT = DATEDIFF(DAY, @NgayLapHopDong, @NgayTinhToan);
        IF @SoNgay = 0 SET @SoNgay = 1; -- Cầm và chuộc trong ngày vẫn tính 1 ngày lãi
        
        SET @TongLai = @SoTienVayGoc * @LaiSuatNgay * @SoNgay;
    END
    ELSE
    BEGIN
        -- ÁP DỤNG LÃI KÉP (Vì đã quá HanChotMot)
        -- A. Tính Lãi đơn tích lũy cho đến HanChotMot
        DECLARE @SoNgayDon INT = DATEDIFF(DAY, @NgayLapHopDong, @HanChotMot);
        IF @SoNgayDon = 0 SET @SoNgayDon = 1;
        DECLARE @LaiDonTichLuy DECIMAL(18,2) = @SoTienVayGoc * @LaiSuatNgay * @SoNgayDon;
        
        -- B. Gộp (Gốc cũ + Lãi đơn) thành Gốc mới
        DECLARE @GocMoi DECIMAL(18,2) = @SoTienVayGoc + @LaiDonTichLuy;
        
        -- C. Tính Lãi kép bằng hàm lũy thừa: GốcMới * (1 + LãiSuất)^SốNgày
        DECLARE @SoNgayKep INT = DATEDIFF(DAY, @HanChotMot, @NgayTinhToan);
        DECLARE @TongTienSauLaiKep DECIMAL(18,2) = @GocMoi * POWER(1.0 + @LaiSuatNgay, @SoNgayKep);
        
        -- D. Tính ra số tiền Lãi thuần túy
        SET @TongLai = @TongTienSauLaiKep - @SoTienVayGoc;
    END

    -- ==========================================
    -- BƯỚC 2: TRỪ ĐI SỐ TIỀN KHÁCH ĐÃ TRẢ GÓP (NẾU CÓ)
    -- ==========================================
    DECLARE @TongTienDaTra DECIMAL(18,2) = 0;
    
    SELECT @TongTienDaTra = ISNULL(SUM(SoTienTra), 0) 
    FROM NhatKyGiaoDich 
    WHERE MaHopDong = @MaHopDong 
      AND CAST(NgayGiaoDich AS DATE) <= @NgayTinhToan;

    -- Logic: Trừ tiền trả góp vào Lãi trước, thừa mới đập vào Gốc
    DECLARE @LaiConLai DECIMAL(18,2) = @TongLai - @TongTienDaTra;
    DECLARE @GocConLai DECIMAL(18,2) = @SoTienVayGoc;

    IF @LaiConLai < 0
    BEGIN
        -- Khách trả dư tiền lãi, phần dư (mang dấu âm) sẽ gộp vào làm giảm Gốc
        SET @GocConLai = @SoTienVayGoc + @LaiConLai; 
        SET @LaiConLai = 0;
    END
    
    IF @GocConLai < 0 SET @GocConLai = 0;

    -- ==========================================
    -- BƯỚC 3: TRẢ VỀ TỔNG CÔNG NỢ (GỐC + LÃI)
    -- ==========================================
    RETURN @GocConLai + @LaiConLai;
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a9782b58-7063-4c83-a82a-f0eff5a09b86" />
- Hàm 2: Tính tổng toàn bộ nợ của 1 khách hàng vì có thể họ đang có 2-3 hợp đồng chưa thanh toán hết

```
IF OBJECT_ID('FnTinhTongNoKhachHang', 'FN') IS NOT NULL DROP FUNCTION FnTinhTongNoKhachHang;
GO

CREATE FUNCTION FnTinhTongNoKhachHang (
    @MaKhachHang INT,
    @NgayTinhToan DATE
)
RETURNS DECIMAL(18,2)
AS
BEGIN
    DECLARE @TongNợKhachHang DECIMAL(18,2) = 0;

    -- Quét tất cả hợp đồng chưa thanh toán xong của khách, gọi hàm 1 để cộng dồn
    SELECT @TongNợKhachHang = ISNULL(SUM(dbo.FnTinhTienHopDong(MaHopDong, @NgayTinhToan)), 0)
    FROM HopDong
    WHERE MaKhachHang = @MaKhachHang 
      AND TrangThaiHopDong IN (N'Đang vay', N'Quá hạn (nợ xấu)', N'Đang trả góp');

    RETURN @TongNợKhachHang;
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0a00b164-ac6b-4135-b6b0-0e37134ea045" />
- Chương trình kiểm tra 2 hàm tính lãi ở trên

Giải thích kết quả chương trình: 
- 1. Thông số ban đầu:
    - Tiền gốc: 10.000.000đ
    - Ngày vay: 09/05/2026
    - Hạn chót 1 (HanChotMot): 01/06/2026
    - Lãi suất: 5.000đ / 1 triệu / 1 ngày $\rightarrow$ Tức là 0,5%/ngày (0.005).
- 2. Giai đoạn 1: Tính Lãi Đơn (Từ 09/05 đến 01/06)
    - Khoảng thời gian này là 23 ngày.
    - Tiền lãi đơn = 10.000.000 x 0.005 x 23 ngày = 1.150.000đ
- 3. Giai đoạn 2: Tính Lãi Kép phạt trễ hạn (Từ 01/06 đến 10/06)
    - Vì ngày test là 10/06, đã vượt quá HanChotMot, hệ thống tự động bật "Công tắc lãi kép" theo đúng nghiệp vụ đề bài:
    - Gộp gốc mới: Hệ thống lấy (Gốc cũ 10tr) + (Lãi đơn 1.150.000đ) = 11.150.000đ. Đây chính là số nợ bị chốt vào ngày 01/06.
    - Số ngày chịu lãi kép: Từ 01/06 đến 10/06 là 9 ngày.
    - Công thức lãi kép (Lũy thừa): Tổng tiền = Gốc Mới x (1 + Lãi Suất)^Số Ngày
  
--> Tổng tiền = 11.150.000 x (1 + 0.005)^9 = 11.150.000 x 1.0459... ~ 11.661.903đ
































































































