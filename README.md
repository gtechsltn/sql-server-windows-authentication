# Sql Server Windows Authentication

# Lưu ý quan trọng
+ Sử dụng tên miền: Luôn sử dụng cú pháp DOMAIN\Username hoặc DOMAIN\GroupName để chỉ định tài khoản Windows.
+ Nguyên tắc đặc quyền tối thiểu: Chỉ cấp những quyền cần thiết để User thực hiện công việc, tránh cấp quyền quá rộng như sysadmin hoặc db_owner nếu không cần thiết.
+ Sử dụng nhóm: Thay vì cấp quyền cho từng tài khoản người dùng, hãy tạo các nhóm Windows (ví dụ: "IT Department", "Sales Team") và cấp quyền cho các nhóm này. Sau đó, chỉ cần thêm người dùng vào nhóm tương ứng trên Windows, việc quản lý sẽ trở nên dễ dàng và hiệu quả hơn.

# Ưu điểm của đoạn script bên dưới
+ Chạy nhiều lần không báo lỗi.
+ Nếu login/user mismatch (hay gặp khi restore DB) thì tự sửa.
+ Chỉ cần đổi UserDb và MANH\ADMIN theo môi trường.

```
USE [master];
GO

-- 1. Tạo LOGIN ở cấp Server nếu chưa có
IF NOT EXISTS (
    SELECT 1 FROM sys.server_principals WHERE name = N'MANH\ADMIN'
)
BEGIN
    CREATE LOGIN [MANH\ADMIN] FROM WINDOWS;
    PRINT 'Created LOGIN [MANH\ADMIN]';
END
ELSE
BEGIN
    PRINT 'LOGIN [MANH\ADMIN] already exists';
END
GO

-- 2. Vào database cụ thể
USE [UserDb];
GO

-- 3. Tạo USER trong database nếu chưa có
IF NOT EXISTS (
    SELECT 1 FROM sys.database_principals WHERE name = N'MANH\ADMIN'
)
BEGIN
    CREATE USER [MANH\ADMIN] FOR LOGIN [MANH\ADMIN];
    PRINT 'Created USER [MANH\ADMIN]';
END
ELSE
BEGIN
    -- Nếu user đã tồn tại nhưng chưa map đúng login thì sửa mapping
    ALTER USER [MANH\ADMIN] WITH LOGIN = [MANH\ADMIN];
    PRINT 'USER [MANH\ADMIN] already exists -> remapped to LOGIN [MANH\ADMIN]';
END
GO

-- 4. Gán quyền cho user (ở đây ví dụ: đọc + ghi)
EXEC sp_addrolemember N'db_datareader', N'MANH\ADMIN';
EXEC sp_addrolemember N'db_datawriter', N'MANH\ADMIN';
PRINT 'Granted db_datareader + db_datawriter to [MANH\ADMIN]';
GO
```
