<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/93ff49ec-14df-4b95-9d86-a91905c8b480" /># MÔN HỌC: HỆ QUẢN TRỊ CƠ SỞ DỮ LIỆU - ThS.ĐỖ DUY CỐP
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

---

### EVENT 3: XỬ LÝ TRẢ NỢ VÀ HOÀN TRẢ TÀI SẢN

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/61043d51-9e21-46e1-a410-b2fe20ae47fa" />
- Store Procedure xử lý khi khách mang tiền đến: 

**Đoạn 1: Khai báo tham số và Lớp khiên bảo vệ**
```
-- Khai báo các biến đầu vào
    -- ĐOẠN 1: KIỂM TRA CHẶN GIAO DỊCH
    IF EXISTS (SELECT 1 FROM TaiSan WHERE MaHopDong = @MaHopDong AND TrangThaiTaiSan = N'Đã bán thanh lý')
    BEGIN
        PRINT N' TỪ CHỐI GIAO DỊCH: Tài sản của hợp đồng này đã bị mang đi thanh lý (quá hạn). Không thể thu tiền hay trả đồ!';
        RETURN; -- Lập tức thoát khỏi Procedure, hủy toàn bộ thao tác bên dưới
    END
```

Chương trình dùng lệnh IF EXISTS để kiểm tra trạng thái tài sản. Nếu có bất kỳ món đồ nào đã bị tiệm đem đi bán (Đã bán thanh lý), hệ thống sẽ khóa mõm giao dịch này lại bằng lệnh RETURN, nhân viên sẽ không thể thu thêm tiền của khách hay xuất đồ ra nữa.

**Đoạn 2: Tính toán công nợ và Lưu sổ sách**
```
    -- ĐOẠN 2: TÍNH TỔNG NỢ & LƯU VẾT VÀO NHẬT KÝ
    -- Gọi lại Hàm số 1 ở Event 2 để xem tính đến hôm nay khách nợ tổng bao nhiêu (Gốc + Lãi)
    DECLARE @TongNoHienTai DECIMAL(18,2) = dbo.FnTinhTienHopDong(@MaHopDong, GETDATE());
    
    -- Lệnh BẮT BUỘC: Ghi nhận lịch sử dòng tiền vào bảng NhatKyGiaoDich
    INSERT INTO NhatKyGiaoDich (MaHopDong, NgayGiaoDich, SoTienTra, NguoiThuTien, GhiChu)
    VALUES (@MaHopDong, GETDATE(), @SoTienTra, @NguoiThuTien, N'Khách trả tiền mặt');
    
    -- Tính ra số dư nợ còn lại sau khi khách ném tiền vào
    DECLARE @DuNoConLai DECIMAL(18,2) = @TongNoHienTai - @SoTienTra;
```

Thay vì tự tính lại tiền lãi ở đây rất dài dòng, em tái sử dụng (reuse) Function ở Event 2. Sau đó, tiền khách đưa BẮT BUỘC phải INSERT ngay vào bảng NhatKyGiaoDich trước khi làm bất cứ việc gì khác, đảm bảo dòng tiền luôn được lưu vết.

**Đoạn 3: Cập nhật Trạng thái Hợp đồng (Trả đứt vs Trả góp)**
```

    -- CẬP NHẬT TRẠNG THÁI HỢP ĐỒNG

    IF @DuNoConLai <= 0 
    BEGIN
        -- Khách đưa dư tiền hoặc vừa đủ -> Xóa nợ, xuất kho trả hết đồ
        SET @DuNoConLai = 0;
        UPDATE HopDong SET TrangThaiHopDong = N'Đã thanh toán đủ' WHERE MaHopDong = @MaHopDong;
        UPDATE TaiSan SET TrangThaiTaiSan = N'Đã trả khách' WHERE MaHopDong = @MaHopDong AND TrangThaiTaiSan = N'Đang cầm cố';
        PRINT N' Khách đã tất toán TOÀN BỘ nợ. Đã cập nhật HD và xuất toàn bộ tài sản!';
    END
    ELSE
    BEGIN
        -- Khách vẫn còn nợ -> Chỉ cập nhật trạng thái
        UPDATE HopDong SET TrangThaiHopDong = N'Đang trả góp' WHERE MaHopDong = @MaHopDong;
        PRINT N' Đã thu tiền gốc/lãi. Khách chuyển sang "Đang trả góp". Dư nợ còn lại: ' + CAST(@DuNoConLai AS NVARCHAR(50));
```

Nếu khách trả sạch tiền (@DuNoConLai <= 0), hệ thống của em dùng lệnh UPDATE để tự động đổi trạng thái toàn bộ đồ đang cầm cố thành Đã trả khách. Tiệm không cần phải xuất từng món thủ công.

**Đoạn 4: Rút tài sản từng phần**
```
        -- ĐIỀU KIỆN RÚT TÀI SẢN (NGHIỆP VỤ CỐT LÕI)
        IF @MaTaiSanMuonRut IS NOT NULL
        BEGIN
            -- Quét kho: Cộng tổng giá trị các món đồ CÒN LẠI (bỏ qua món khách đang xin rút)
            DECLARE @GiaTriKhoConLai DECIMAL(18,2) = 0;
            SELECT @GiaTriKhoConLai = ISNULL(SUM(GiaTriDinhGia), 0)
            FROM TaiSan
            WHERE MaHopDong = @MaHopDong 
              AND TrangThaiTaiSan = N'Đang cầm cố' 
              AND MaTaiSan != @MaTaiSanMuonRut;

            -- Quy tắc tử huyệt: Giá trị giữ lại >= Nợ còn lại
            IF @GiaTriKhoConLai >= @DuNoConLai
            BEGIN
                UPDATE TaiSan SET TrangThaiTaiSan = N'Đã trả khách' WHERE MaTaiSan = @MaTaiSanMuonRut;
                PRINT N'ĐỦ ĐIỀU KIỆN! Giá trị kho còn lại (' + CAST(@GiaTriKhoConLai AS NVARCHAR(50)) + N') an toàn so với khoản nợ. Đã xuất đồ!';
            END
            ELSE
            BEGIN
                PRINT N'TỪ CHỐI XUẤT ĐỒ! Giá trị kho còn lại (' + CAST(@GiaTriKhoConLai AS NVARCHAR(50)) + N') NHỎ HƠN nợ hiện tại (' + CAST(@DuNoConLai AS NVARCHAR(50)) + N'). Rủi ro mất vốn!';
            END
        END
    END -- Kết thúc nhánh ELSE của đoạn 3
```

Chương trình sử dụng hàm SUM với điều kiện MaTaiSan != @MaTaiSanMuonRut để giả định rào lại kho đồ sau khi khách lấy đồ đi. Nếu phần giữ lại đủ bù nợ thì mới kích hoạt UPDATE trả đồ, nếu không hệ thống sẽ chặn đứng và báo 'Rủi ro mất vốn'.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/674dede3-7744-43bc-8a78-ff24060c070d" />
- Demo tình huống khách đến trả 1 phần tiền và xin rút đi 1 món đồ đang cầm cố để sử dụng. Trả trước 5m để rút laptop ra (điện thoại vẫn đang cầm cố), khách đang nợ cả gốc lẫn lãi là 11m6(ở EV2), khách trả trước 5m thì còn 6m6. Hệ thống xác định giá trị tài sản còn lại>Dư nợ --> cho rút laptop ra vì dù có bị quỵt không trả tiền nợ thì chủ tiệm vẫn có thể bán món còn lại đang cầm cố để bù nợ.

---

**Event 4: Truy vấn danh sách nợ xấu (Nợ khó đòi)**
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/8867e094-78f6-485d-afc6-7eab2b096312" />
- Tạo dữ liệu khách hàng thứ 2 trả tiền quá hạn
```
-- TẠO DỮ LIỆU TEST KHÁCH NỢ XẤU
USE QuanLyThuNo_K235480106001;
GO
-- 1. Thêm Khách hàng 
INSERT INTO KhachHang (HoTen, SoDienThoai, SoCanCuocCongDan, DiaChi)
VALUES (N'Trần Văn Bùng', '0988111222', '031099002233', N'Hải Phòng');
DECLARE @MaKH_NoXau INT = SCOPE_IDENTITY(); 
-- 2. Thêm Hợp đồng 
-- Vay từ tháng 2, Hạn chót 1 là 01/03, Hạn chót 2 là 01/04/2026 -> Hôm nay là tháng 5, chắc chắn dính nợ xấu!
INSERT INTO HopDong (MaKhachHang, NgayLapHopDong, SoTienVayGoc, HanChotMot, HanChotHai, TrangThaiHopDong)
VALUES (@MaKH_NoXau, '2026-02-01', 15000000.00, '2026-03-01', '2026-04-01', N'Đang cầm cố');
DECLARE @MaHD_NoXau INT = SCOPE_IDENTITY(); 
-- 3. Thêm Tài sản thế chấp
INSERT INTO TaiSan (MaHopDong, TenTaiSan, GiaTriDinhGia, TrangThaiTaiSan)
VALUES (@MaHD_NoXau, N'Xe máy Honda SH 150i', 40000000.00, N'Đang cầm cố');
PRINT N'Đã tạo thành công dữ liệu Khách Nợ Xấu !';
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5e43db0e-1cd5-49f5-be53-c9ebfe1b2e78" />
- Hàm truy vấn danh sách khách nợ xấu
```
CREATE PROCEDURE SpDanhSachNoXau
AS
BEGIN
    SET NOCOUNT ON;
    
    PRINT N'=== BÁO CÁO DANH SÁCH NỢ XẤU TÍNH ĐẾN HÔM NAY (' + CAST(GETDATE() AS NVARCHAR(20)) + N') ===';

    SELECT 
        KH.MaKhachHang,
        KH.HoTen AS TenKhachHang,
        KH.SoDienThoai,
        KH.SoCanCuocCongDan, -- Đã sửa từ CCCD sang đúng tên trường của bạn
        HD.MaHopDong,
        HD.NgayLapHopDong,   -- Đã sửa từ NgayVay sang đúng tên trường của bạn
        HD.HanChotHai AS NgayHetHanCuoiCung,
        
        -- Tính số ngày đã quá hạn so với hôm nay
        DATEDIFF(DAY, HD.HanChotHai, GETDATE()) AS SoNgayQuaHan,
        
        -- Gọi Function tính nợ, tính đến thời điểm hiện tại
        dbo.FnTinhTienHopDong(HD.MaHopDong, GETDATE()) AS TongNoPhaiThu,
        
        -- Tính tổng giá trị các món đồ CÒN TRONG KHO
        (SELECT ISNULL(SUM(GiaTriDinhGia), 0) 
         FROM TaiSan 
         WHERE MaHopDong = HD.MaHopDong AND TrangThaiTaiSan = N'Đang cầm cố') AS TongGiaTriDoTrongKho,
         
        -- Đánh giá rủi ro dựa trên giá trị tài sản còn lại
        CASE 
            WHEN (SELECT ISNULL(SUM(GiaTriDinhGia), 0) FROM TaiSan WHERE MaHopDong = HD.MaHopDong AND TrangThaiTaiSan = N'Đang cầm cố') >= dbo.FnTinhTienHopDong(HD.MaHopDong, GETDATE())
            THEN N'An toàn (Đủ đồ thanh lý)'
            ELSE N'Nguy hiểm (Nợ vượt giá trị đồ)'
        END AS DanhGiaRuiRo

    FROM HopDong HD
    JOIN KhachHang KH ON HD.MaKhachHang = KH.MaKhachHang
    WHERE 
        -- Điều kiện nợ xấu: Đã quá Hạn Chót 2
        HD.HanChotHai < GETDATE() 
        -- Và hợp đồng vẫn đang trong quá trình thực hiện, chưa kết thúc
        AND HD.TrangThaiHopDong NOT IN (N'Đã thanh toán', N'Đã thanh toán đủ', N'Đã bán thanh lý');
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/12cb3f52-a7ff-4b00-bc0a-3323487134df" />
- Câu lệnh lọc danh sách các khách hàng đang nợ xấu nhưng ở trong ngưỡng an toàn (đủ đồ thanh lý)

---

#### Event 5: Quản lý thanh lý tài sản 

**Nguyên nhân cần sử dụng trigger?:**
Trong ngành cầm đồ, rủi ro lớn nhất không chỉ đến từ khách hàng bùng nợ, mà còn đến từ nhân viên gian lận. Giả sử một khách hàng đến trả nợ 15 triệu. Nhân viên nhận tiền bỏ túi riêng, sau đó lén vào phần mềm XÓA luôn dòng hợp đồng đó đi để phi tang chứng cứ. Lúc này chủ tiệm sẽ mất trắng cả tiền lẫn đồ.

--> Vì vậy, chúng ta cần một Trigger canh gác ở cổng bảng HopDong. Bất cứ ai dùng lệnh DELETE để xóa một hợp đồng đang ở trạng thái Đang cầm cố hoặc Đang trả góp, Trigger sẽ đá bay lệnh đó, báo lỗi và khôi phục lại dữ liệu ngay lập tức.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/647c29f3-ed81-4ca6-88d0-a8350535a1c2" />
- Chương trình chính của trigger bảo mật an toàn dữ liệu

```
CREATE TRIGGER Trg_BaoMat_NganXoaHopDong
ON HopDong
FOR DELETE
AS
BEGIN
    -- có hợp đồng nào đang ở trạng thái hoạt động hay không?
    IF EXISTS (
        SELECT 1 
        FROM deleted 
        WHERE TrangThaiHopDong IN (N'Đang cầm cố', N'Đang trả góp')
    )
    BEGIN
        -- Nếu có, lập tức báo 
        RAISERROR (N'CẢNH BÁO BẢO MẬT: Tuyệt đối không được xóa Hợp đồng đang trong quá trình cầm cố hoặc trả góp! Nếu khách đã tất toán, vui lòng cập nhật trạng thái thành "Đã thanh toán"!', 16, 1);
        
        -- Hủy bỏ hoàn toàn thao tác xóa, trả lại dữ liệu như cũ
        ROLLBACK TRANSACTION;
    END
    ELSE
    BEGIN
        PRINT N'Hợp đồng đã đóng (Đã thanh toán/Đã thanh lý). Cho phép xóa thành công!';
    END
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/be0c956a-f538-487c-8823-b055b03632ef" />
- Demo trường hợp nhân viên của quán lỡ tay/cố tình xóa đi 1 hợp đồng đề làm mục đích cá nhân, lúc này hệ thống sẽ cảnh báo đỏ và in ra trên màn hình hiển thị mà nhân viên đang sử dụng.

---

**4. Các sự kiện bổ sung**
chưa làm phần này trở đi





























































