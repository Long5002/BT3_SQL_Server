# BT3_SQL_Server

<h3>Đề tài: Thiết kế và cài đặt CSDL quản lý cầm đồ </h3>

## Thông tin sinh viên:

* Họ và tên: Nguyễn Việt Long

* MSSV: K235480106047

* Lớp: K59KMT

* Giảng viên hướng dẫn: ThS. Đỗ Duy Cốp

## Nhiệm vụ 1 (file PDF)

## Nhiệm vụ 2 (file Script.sql)

Tạo mới database:

```
Create database QuanLyCamDo_K235480106047;
Go
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/dfb304f0-e0cd-4b4a-9d0c-b32d95d40af5" />

Tạo bảng và khởi tạo dữ liệu:

```
Use QuanLyCamDo_K235480106047;
Go

	-- 1. Tạo bảng Khách hàng
CREATE TABLE KhachHang (
    MaKH INT IDENTITY(1,1) PRIMARY KEY,
    HoTen NVARCHAR(255) NOT NULL,
    SoCCCD VARCHAR(20) UNIQUE NOT NULL, -- Định danh duy nhất
    SoDienThoai VARCHAR(15) UNIQUE NOT NULL,
    DiaChi NVARCHAR(255)
);

-- 2. Tạo bảng Hợp đồng
CREATE TABLE HopDong (
    MaHD INT IDENTITY(1,1) PRIMARY KEY,
    MaKH INT NOT NULL,
    NgayLap DATETIME DEFAULT GETDATE(),
    Deadline1 DATE NOT NULL,           -- Mốc thời gian chuyển từ lãi đơn sang lãi kép
    SoTienGoc DECIMAL(18, 2) NOT NULL,  -- Số tiền vay ban đầu
    TrangThai NVARCHAR(50) DEFAULT N'Đang vay', 
    FOREIGN KEY (MaKH) REFERENCES KhachHang(MaKH) ON DELETE CASCADE
);

-- 3. Tạo bảng Tài sản thế chấp (1 Hợp đồng có nhiều tài sản)
CREATE TABLE TaiSan (
    MaTS INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT NOT NULL,
    TenTaiSan NVARCHAR(255) NOT NULL,
    GiaTriUocTinh DECIMAL(18, 2),       -- Giá trị để so sánh khi trả hàng
    MoTa NVARCHAR(500),
    TrangThai NVARCHAR(50) DEFAULT N'Đang cầm cố',
    NgayCapNhat DATETIME DEFAULT GETDATE(),
    FOREIGN KEY (MaHD) REFERENCES HopDong(MaHD) ON DELETE CASCADE
);

-- 4. Tạo bảng Log (Lưu lịch sử biến động trạng thái và tiền nợ)
CREATE TABLE LogBienDong (
    MaLog INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT NOT NULL,
    NgayBienDong DATETIME DEFAULT CURRENT_TIMESTAMP, -- Thời gian thực
    LoaiBienDong NVARCHAR(100),          -- 'Trả nợ gốc', 'Chốt lãi kép', 'Thanh lý'
    SoTienThayDoi DECIMAL(18, 2),       -- Số tiền biến động (+/-)
    DuNoHienTai DECIMAL(18, 2),          -- Tổng dư nợ thực tế sau biến động
    GhiChu NVARCHAR(500),
    FOREIGN KEY (MaHD) REFERENCES HopDong(MaHD) ON DELETE CASCADE
);
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/f6324f68-206a-4805-8891-04c64479e258" />

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/2b88aa0d-9376-4084-a752-11224798ee3f" />

Sơ đồ ERD

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/679f621c-92df-4963-86ba-3f773b9b846f" />

### Event 1: Đăng kí hợp đồng mới (vay tiền)

Viết Store Procedure tiếp nhận hợp đồng: Lưu thông tin khách hàng, danh sách tài sản (kèm giá trị định giá), số tiền vay gốc và thiết lập 2 mốc Deadline1, Deadline2.

Thêm Deadline 2 vào bảng HopDong:

```
Use QuanLyCamDo_K235480106047;
Go
ALTER TABLE HopDong ADD [Deadline2] DATE NOT NULL;
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/5649280e-a443-4fcd-b831-f84a9761f54f" />

Viết Store Procedure:

```
CREATE TYPE Table_DanhSachTaiSan AS TABLE (
    TenTaiSan NVARCHAR(255),
    GiaTriUocTinh DECIMAL(18, 2),
    MoTa NVARCHAR(500)
);
GO

CREATE PROCEDURE sp_TiepNhanHopDong
    @MaKH INT,
    @SoTienGoc DECIMAL(18, 2),
    @Deadline1 DATE,
    @Deadline2 DATE,
    @DSTaiSan Table_DanhSachTaiSan READONLY
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRANSACTION;
    BEGIN TRY
        -- 1. Lưu thông tin hợp đồng
        INSERT INTO HopDong (MaKH, SoTienGoc, Deadline1, Deadline2, TrangThai)
        VALUES (@MaKH, @SoTienGoc, @Deadline1, @Deadline2, N'Đang vay');

        DECLARE @MaHD_Moi INT = SCOPE_IDENTITY();

        -- 2. Lưu danh sách tài sản
        INSERT INTO TaiSan (MaHD, TenTaiSan, GiaTriUocTinh, MoTa)
        SELECT @MaHD_Moi, TenTaiSan, GiaTriUocTinh, MoTa FROM @DSTaiSan;

        -- 3. Ghi Log khởi tạo
        INSERT INTO LogBienDong (MaHD, LoaiBienDong, SoTienThayDoi, DuNoHienTai, GhiChu)
        VALUES (@MaHD_Moi, N'Mở hợp đồng', @SoTienGoc, @SoTienGoc, N'Bắt đầu hợp đồng vay');

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/8784aa19-6158-457b-a813-e0b37db397d5" />

Test Event 1:

```
USE QuanLyCamDo_K235480106047
GO
INSERT INTO KhachHang (HoTen, SoCCCD, SoDienThoai, DiaChi)
VALUES (N'Nguyễn Văn Linh', '012010884901', '0901237362', N'123 Đường Hai Bà Trưng, Hà Nội');

-- Lấy MaKH vừa chèn để dùng cho bước sau
DECLARE @MaKH_Test INT = SCOPE_IDENTITY();

-- 2. Khai báo biến bảng để chứa danh sách tài sản (Dùng Type đã tạo ở Event 1)
DECLARE @DS_TaiSan_Mẫu Table_DanhSachTaiSan;

INSERT INTO @DS_TaiSan_Mẫu (TenTaiSan, GiaTriUocTinh, MoTa)
VALUES 
(N'Xe máy Honda Vision 2023', 35000000, N'Xe mới 95%, đầy đủ giấy tờ'),
(N'Điện thoại iPhone 14 Pro', 20000000, N'Bản 128GB, màu tím');

-- 3. Thiết lập các mốc thời gian
DECLARE @DL1 DATE = DATEADD(DAY, 30, GETDATE()); -- Deadline 1 sau 30 ngày
DECLARE @DL2 DATE = DATEADD(DAY, 45, GETDATE()); -- Deadline 2 sau 45 ngày

-- 4. Thực thi Store Procedure
EXEC sp_TiepNhanHopDong 
    @MaKH = @MaKH_Test, 
    @SoTienGoc = 15000000, 
    @Deadline1 = @DL1, 
    @Deadline2 = @DL2, 
    @DSTaiSan = @DS_TaiSan_Mẫu;

-- 5. Kiểm tra kết quả trong các bảng
PRINT '--- Dữ liệu trong bảng HopDong ---';
SELECT * FROM HopDong WHERE MaKH = @MaKH_Test;

PRINT '--- Dữ liệu trong bảng TaiSan ---';
SELECT ts.* FROM TaiSan ts 
JOIN HopDong hd ON ts.MaHD = hd.MaHD 
WHERE hd.MaKH = @MaKH_Test;

PRINT '--- Dữ liệu trong bảng LogBienDong ---';
SELECT l.* FROM LogBienDong l
JOIN HopDong hd ON l.MaHD = hd.MaHD     
WHERE hd.MaKH = @MaKH_Test;
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/60f991b7-fb40-48e0-bf9b-ea836f11d940" />

### Event 2: Tính toán công nợ thời gian thực

Viết 2 Function:

- Function fn_CalcMoneyTransaction: để tính số tiền phải trả của TransactionID này cho đến ngày TargetDate

- Function fn_CalcMoneyContract: để tính tổng số tiền khách(ContractID) phải trả (Gốc + Lãi đơn + Lãi kép) tính đến ngày TargetDate

```
USE QuanLyCamDo_K235480106047
GO
CREATE FUNCTION fn_CalcMoneyTransaction(@MaLog INT, @TargetDate DATETIME)
RETURNS DECIMAL(18, 2)
AS
BEGIN
    DECLARE @MaHD INT, @DuNoTaiThoiDiemDo DECIMAL(18, 2), @NgayBienDong DATETIME;
    
    -- Lấy thông tin từ bảng Log
    SELECT @MaHD = MaHD, @DuNoTaiThoiDiemDo = DuNoHienTai, @NgayBienDong = NgayBienDong 
    FROM LogBienDong WHERE MaLog = @MaLog;

    -- Nếu ngày mục tiêu trước ngày biến động, kết quả không hợp lệ (hoặc trả về 0)
    IF (@TargetDate < @NgayBienDong) RETURN 0;

    -- Tính toán dựa trên dư nợ tại thời điểm đó (áp dụng lãi suất đơn 0.5%/ngày)
    DECLARE @SoNgay INT = DATEDIFF(DAY, @NgayBienDong, @TargetDate);
    RETURN @DuNoTaiThoiDiemDo + (@DuNoTaiThoiDiemDo * 0.005 * @SoNgay);
END;
GO

CREATE FUNCTION fn_CalcMoneyContract(@MaHD INT, @TargetDate DATE)
RETURNS DECIMAL(18, 2)
AS
BEGIN
    DECLARE @Goc DECIMAL(18, 2), @NgayLap DATE, @D1 DATE;
    DECLARE @TongTien DECIMAL(18, 2);
    DECLARE @CurrentDate DATE;

    -- 1. Lấy thông tin hợp đồng
    SELECT @Goc = SoTienGoc, @NgayLap = CAST(NgayLap AS DATE), @D1 = Deadline1 
    FROM HopDong WHERE MaHD = @MaHD;

    IF @TargetDate <= @NgayLap RETURN @Goc;

    -- 2. TÍNH LÃI ĐƠN (Từ NgayLap đến D1 hoặc TargetDate)
    DECLARE @EndDateForSimpleInterest DATE = CASE WHEN @TargetDate < @D1 THEN @TargetDate ELSE @D1 END;
    DECLARE @SoNgayLaiDon INT = DATEDIFF(DAY, @NgayLap, @EndDateForSimpleInterest);
    
    -- Tiền nợ sau giai đoạn lãi đơn
    SET @TongTien = @Goc + (@Goc * 0.005 * @SoNgayLaiDon);

    -- 3. TÍNH LÃI KÉP (Nếu TargetDate vượt quá D1)
    IF @TargetDate > @D1
    BEGIN
        SET @CurrentDate = DATEADD(DAY, 1, @D1);
        
        -- Dùng vòng lặp để tính lãi kép theo từng ngày
        WHILE @CurrentDate <= @TargetDate
        BEGIN
            -- Lãi mỗi ngày là 0.5% dựa trên tổng số tiền (Gốc + Lãi cũ) của ngày hôm trước
            SET @TongTien = @TongTien + (@TongTien * 0.005);
            SET @CurrentDate = DATEADD(DAY, 1, @CurrentDate);
        END
    END

    RETURN ROUND(@TongTien, 2);
END;
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/248b2d11-ade0-4db7-a8fb-5c65294bc0ad" />

Test Event 2:

- Function fn_CalcMoneyContract

```
USE QuanLyCamDo_K235480106047
GO
-- Giả sử MaHD = 1, tính nợ tại ngày hiện tại
SELECT 
    MaHD, 
    SoTienGoc, 
    NgayLap, 
    Deadline1,
    dbo.fn_CalcMoneyContract(MaHD, CAST(GETDATE() AS DATE)) AS TongNoHienTai
FROM HopDong
WHERE MaHD = 1;

-- Test thử một ngày xa trong tương lai để thấy lãi kép nhảy số
SELECT dbo.fn_CalcMoneyContract(1, '2026-12-31') AS NoTuongLai;
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/5fefd6be-22bf-4f3f-983f-fc0635c4fbcd" />

- Function fn_CalcMoneyTransaction

```
USE QuanLyCamDo_K235480106047
GO

-- Chúng ta giả lập một lần khách đến trả bớt tiền (biến động dư nợ)
INSERT INTO LogBienDong (MaHD, NgayBienDong, LoaiBienDong, SoTienThayDoi, DuNoHienTai, GhiChu)
VALUES (1, DATEADD(DAY, -5, GETDATE()), N'Khách trả nợ gốc', 2000000, 8000000, N'Dư nợ còn lại 8tr từ 5 ngày trước');

-- Lấy MaLog vừa tạo để test
DECLARE @MaLogTest INT = SCOPE_IDENTITY();
PRINT 'MaLog vua tao la: ' + CAST(@MaLogTest AS VARCHAR(10));
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/51591712-3bc8-4bb6-8545-9c87e785f77e" />

```
DECLARE @MaLogTest INT;
-- Lấy ID của dòng Log cuối cùng của MaHD = 1 để test
SELECT TOP 1 @MaLogTest = MaLog FROM LogBienDong WHERE MaHD = 1 ORDER BY NgayBienDong DESC;

-- Thực hiện truy vấn kiểm tra
SELECT 
    L.MaLog,
    L.MaHD,
    L.NgayBienDong AS [Ngày ghi nhận Log],
    L.DuNoHienTai AS [Dư nợ gốc tại mốc đó],
    GETDATE() AS [Ngày mục tiêu (TargetDate)],
    DATEDIFF(DAY, L.NgayBienDong, GETDATE()) AS [Số ngày trôi qua kể từ Log],
    -- Gọi hàm test
    dbo.fn_CalcMoneyTransaction(@MaLogTest, GETDATE()) AS [Tổng tiền dự kiến (Gốc + Lãi đơn)]
FROM LogBienDong L
WHERE L.MaLog = @MaLogTest;
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/ee26e494-1489-475f-aff8-3bb55fc2feb1" />

### Event 3: Xử lý trả nợ và hoàn trả tài sản

```
CREATE PROCEDURE sp_XuLyTraNo_V2
    @MaHD INT,
    @SoTienKhachDua DECIMAL(18, 2),
    @NguoiThu NVARCHAR(100) = N'Nhân viên hệ thống'
AS
BEGIN
    SET NOCOUNT ON;

    -- 1. KHAI BÁO CÁC BIẾN CẦN THIẾT
    DECLARE @TongNoHienTai DECIMAL(18, 2);
    DECLARE @DuNoSauKhiTra DECIMAL(18, 2);
    DECLARE @TrangThaiHD NVARCHAR(50);

    -- 2. KIỂM TRA TRẠNG THÁI THANH LÝ
    -- Nếu có bất kỳ tài sản nào trong hợp đồng đã chuyển sang 'Đã bán thanh lý'
    IF EXISTS (SELECT 1 FROM TaiSan WHERE MaHD = @MaHD AND TrangThai = N'Đã bán thanh lý')
    BEGIN
        PRINT N'THÔNG BÁO: Tài sản của hợp đồng này đã bị thanh lý. Không thu tiền, không trả đồ!';
        RETURN;
    END

    -- 3. TÍNH TỔNG NỢ THỰC TẾ (Gọi hàm từ Event 2)
    SET @TongNoHienTai = dbo.fn_CalcMoneyContract(@MaHD, GETDATE());
    SET @DuNoSauKhiTra = @TongNoHienTai - @SoTienKhachDua;

    -- Đảm bảo không trả dư tiền (nếu cần logic này)
    IF @DuNoSauKhiTra < 0 SET @DuNoSauKhiTra = 0;

    -- 4. CẬP NHẬT TRẠNG THÁI HỢP ĐỒNG & TÀI SẢN
    IF @DuNoSauKhiTra = 0
    BEGIN
        -- Trường hợp trả hết nợ
        UPDATE HopDong 
        SET TrangThai = N'Đã thanh toán đủ' 
        WHERE MaHD = @MaHD;

        UPDATE TaiSan 
        SET TrangThai = N'Đã trả khách', NgayCapNhat = GETDATE() 
        WHERE MaHD = @MaHD;

        PRINT N'Xác nhận: Khách đã thanh toán đủ. Đã cập nhật trạng thái trả hết đồ.';
    END
    ELSE
    BEGIN
        -- Trường hợp trả một phần
        UPDATE HopDong 
        SET TrangThai = N'Đang trả góp' 
        WHERE MaHD = @MaHD;

        PRINT N'Xác nhận: Khách trả một phần. Dư nợ còn lại: ' + CAST(@DuNoSauKhiTra AS NVARCHAR(50));
    END

    -- 5. GHI NHẬN VÀO LOG (Audit Log)
    INSERT INTO LogBienDong (MaHD, NgayBienDong, LoaiBienDong, SoTienThayDoi, DuNoHienTai, GhiChu)
    VALUES (
        @MaHD, 
        GETDATE(), 
        N'Khách trả nợ', 
        @SoTienKhachDua, 
        @DuNoSauKhiTra, 
        N'Người thu: ' + @NguoiThu + N'. Tổng nợ trước khi trả: ' + CAST(@TongNoHienTai AS NVARCHAR(50))
    );

    -- 6. ĐƯA RA DANH SÁCH GỢI Ý TRẢ LẠI TÀI SẢN
    -- Điều kiện: Giá trị tài sản còn lại (các món chưa trả) >= Dư nợ còn lại
    -- Ở đây ta liệt kê các món tài sản mà nếu trả món đó cho khách, 
    -- tổng giá trị các món còn giữ lại vẫn đủ bao phủ khoản nợ.
    
    PRINT N'--- DANH SÁCH GỢI Ý TÀI SẢN CÓ THỂ TRẢ LẠI ---';
    
    DECLARE @TongGiaTriTaiSanHienCo DECIMAL(18, 2);
    SELECT @TongGiaTriTaiSanHienCo = SUM(GiaTriUocTinh) 
    FROM TaiSan WHERE MaHD = @MaHD AND TrangThai = N'Đang cầm cố';

    SELECT 
        MaTS, 
        TenTaiSan, 
        GiaTriUocTinh,
        (@TongGiaTriTaiSanHienCo - GiaTriUocTinh) AS GiaTriConLaiNeuTraMonNay,
        @DuNoSauKhiTra AS DuNoHienTai
    FROM TaiSan
    WHERE MaHD = @MaHD 
      AND TrangThai = N'Đang cầm cố'
      AND (@TongGiaTriTaiSanHienCo - GiaTriUocTinh) >= @DuNoSauKhiTra;

END;
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/f3930db9-989a-405d-bd5e-cb7c074fc266" />

Test Event 3:

```
-- Giả sử MaHD = 1, khách trả 5 triệu
EXEC sp_XuLyTraNo_V2 
    @MaHD = 1, 
    @SoTienKhachDua = 5000000, 
    @NguoiThu = N'Nguyễn Thu Ngân';

-- Kiểm tra lại Log để xem dòng tiền
SELECT * FROM LogBienDong WHERE MaHD = 1 ORDER BY NgayBienDong DESC;
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/0c5fd896-a936-4e6a-bb01-0abdf415ffc5" />

### Event 4: Truy vấn danh sách nợ xấu (Nợ khó đòi)

```
-- Truy vấn danh sách khách hàng nợ xấu (Quá Deadline 1)
SELECT 
    kh.HoTen AS [Tên KH],
    kh.SoDienThoai AS [Số điện thoại],
    hd.SoTienGoc AS [Số tiền vay gốc],
    DATEDIFF(DAY, hd.Deadline1, GETDATE()) AS [Số ngày quá hạn],
    
    -- Tổng tiền phải trả tính đến thời điểm hiện tại
    dbo.fn_CalcMoneyContract(hd.MaHD, CAST(GETDATE() AS DATE)) AS [Tổng tiền phải trả hiện tại],
    
    -- Tổng tiền phải trả sau 1 tháng nữa (Dự báo nợ dựa trên lãi kép)
    dbo.fn_CalcMoneyContract(hd.MaHD, CAST(DATEADD(MONTH, 1, GETDATE()) AS DATE)) AS [Tổng tiền phải trả sau 1 tháng]

FROM HopDong hd
JOIN KhachHang kh ON hd.MaKH = kh.MaKH
WHERE 
    hd.Deadline1 < CAST(GETDATE() AS DATE) -- Đã quá mốc lãi đơn
    AND hd.TrangThai NOT IN (N'Đã thanh toán đủ', N'Đã thanh lý tài sản') -- Chưa tất toán
    AND hd.TrangThai <> N'Đã trả khách'; -- Tài sản vẫn còn ở tiệm hoặc chưa xử lý xong
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/e2bffc7d-0a34-4188-854d-e8b856ef42d1" />

Dùng View để dễ dàng truy xuất

```
CREATE VIEW vw_BaoCaoNoXau AS
SELECT 
    kh.HoTen,
    kh.SoDienThoai,
    hd.SoTienGoc,
    hd.Deadline1,
    DATEDIFF(DAY, hd.Deadline1, GETDATE()) AS SoNgayQuaHan,
    dbo.fn_CalcMoneyContract(hd.MaHD, GETDATE()) AS TongNoHienTai,
    dbo.fn_CalcMoneyContract(hd.MaHD, DATEADD(MONTH, 1, GETDATE())) AS DuKienNoThangSau
FROM HopDong hd
JOIN KhachHang kh ON hd.MaKH = kh.MaKH
WHERE hd.Deadline1 < GETDATE() 
  AND hd.TrangThai NOT IN (N'Đã thanh toán đủ', N'Đã thanh lý tài sản');
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/910d2d14-a5fc-48f5-83fb-f8e0b09398e2" />

Test Event 4:

```
SELECT * FROM vw_BaoCaoNoXau;
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/78919e5b-190d-4740-b221-c79d30bcae6a" />

### Event 5: Quản lý thanh lý tài sản

- Trigger 1: Chuyển hợp đồng sang "quá hạn (nợ xấu)"

```
CREATE TRIGGER trg_CheckDeadline1
ON HopDong
AFTER UPDATE, INSERT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Cập nhật trạng thái hợp đồng dựa trên Deadline 1
    UPDATE HopDong
    SET TrangThai = N'Quá hạn (nợ xấu)'
    FROM HopDong hd
    INNER JOIN inserted i ON hd.MaHD = i.MaHD
    WHERE hd.TrangThai = N'Đang vay' 
      AND GETDATE() > hd.Deadline1;
END;
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/b3daa533-77af-4b71-ad5e-919b4dd492a4" />

-Trigger 2: Chuyển tài sản sang "sẵn sàng thanh lý"

```
CREATE TRIGGER trg_CheckDeadline2
ON HopDong
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Nếu Hợp đồng quá Deadline 2, cập nhật trạng thái Tài sản
    UPDATE TaiSan
    SET TrangThai = N'Sẵn sàng thanh lý',
        NgayCapNhat = GETDATE()
    FROM TaiSan ts
    INNER JOIN inserted i ON ts.MaHD = i.MaHD
    WHERE i.TrangThai = N'Quá hạn (nợ xấu)' 
      AND GETDATE() > i.Deadline2
      AND ts.TrangThai = N'Đang cầm cố';
END;
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/82903bf0-998f-4be7-8ac5-58e8497bb114" />

- Trigger 3: Chuyển tài sản sang "đã bán thanh lý"

```
CREATE TRIGGER trg_ConfirmThanhLyTaiSan
ON HopDong
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Kiểm tra nếu cột TrangThai của HopDong vừa được đổi thành 'Đã thanh lý'
    IF EXISTS (SELECT 1 FROM inserted i JOIN deleted d ON i.MaHD = d.MaHD 
               WHERE i.TrangThai = N'Đã thanh lý' AND d.TrangThai <> N'Đã thanh lý')
    BEGIN
        UPDATE TaiSan
        SET TrangThai = N'Đã bán thanh lý',
            NgayCapNhat = GETDATE()
        FROM TaiSan ts
        INNER JOIN inserted i ON ts.MaHD = i.MaHD
        WHERE i.TrangThai = N'Đã thanh lý'
          AND ts.TrangThai IN (N'Đang cầm cố', N'Sẵn sàng thanh lý');
          
        PRINT N'Hệ thống đã tự động chuyển toàn bộ tài sản sang trạng thái: Đã bán thanh lý';
    END
END;
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/b1b7b09b-f95f-4b38-8e41-456f8961e11a" />

### Sự kiện gia hạn hợp đồng

```
CREATE PROCEDURE sp_GiaHanHopDong
    @MaHD INT,
    @NgayDeadline1_Moi DATE,
    @NgayDeadline2_Moi DATE
AS
BEGIN
    DECLARE @TongNoHienTai DECIMAL(18,2) = dbo.fn_CalcMoneyContract(@MaHD, GETDATE());
    DECLARE @Goc DECIMAL(18,2) = (SELECT SoTienGoc FROM HopDong WHERE MaHD = @MaHD);
    DECLARE @TienLaiPhaiDong DECIMAL(18,2) = @TongNoHienTai - @Goc;

    -- 1. Yêu cầu thanh toán tiền lãi trước khi gia hạn
    PRINT N'Tiền lãi khách cần đóng để gia hạn: ' + CAST(@TienLaiPhaiDong AS NVARCHAR(50));

    -- 2. Cập nhật Deadline và reset ngày lập (để tính lại lãi đơn từ mốc này)
    UPDATE HopDong
    SET Deadline1 = @NgayDeadline1_Moi,
        Deadline2 = @NgayDeadline2_Moi,
        NgayLap = GETDATE(),
        TrangThai = N'Đang vay'
    WHERE MaHD = @MaHD;

    -- 3. Ghi Log
    INSERT INTO LogBienDong (MaHD, LoaiBienDong, SoTienThayDoi, DuNoHienTai, GhiChu)
    VALUES (@MaHD, N'Gia hạn hợp đồng', @TienLaiPhaiDong, @Goc, N'Đã đóng lãi và dời Deadline');
END;
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/72f83b56-05bf-4690-a6ce-d38bbb2d6355" />

