# Sql Server Windows Authentication

Khi cấp quyền trên SQL Server cho tài khoản Windows, bạn có thể thực hiện theo các bước sau:

# 1. Tạo Login cho tài khoản Windows
Trước hết, bạn cần tạo một **Login** (**đăng nhập**) trên SQL Server cho tài khoản Windows đó. Login này sẽ là cầu nối giữa tài khoản hệ điều hành và cơ sở dữ liệu.

```
-- Dành cho tài khoản người dùng Windows
CREATE LOGIN [DOMAIN\Username] FROM WINDOWS;

-- Dành cho nhóm người dùng Windows
CREATE LOGIN [DOMAIN\GroupName] FROM WINDOWS;
```

+ **DOMAIN\Username**: Thay thế bằng tên miền và tên tài khoản người dùng Windows.
+ **DOMAIN\GroupName**: Thay thế bằng tên miền và tên nhóm người dùng Windows.

# 2. Tạo User trong cơ sở dữ liệu
Sau khi tạo Login, bạn cần tạo một **User** (**người dùng**) trong cơ sở dữ liệu mà bạn muốn cấp quyền. User này sẽ được liên kết với Login đã tạo ở bước 1.

```
-- Chọn cơ sở dữ liệu bạn muốn tạo người dùng
USE [DatabaseName];
GO

-- Tạo user từ login
CREATE USER [Username] FOR LOGIN [DOMAIN\Username];
```

+ **DatabaseName**: Tên cơ sở dữ liệu bạn muốn cấp quyền (ví dụ: QLBH).

+ **Username**: Tên người dùng trong cơ sở dữ liệu (nên đặt giống tên tài khoản Windows để dễ quản lý).

# 3. Cấp quyền cho User
Sau khi tạo User, bạn có thể cấp các quyền cụ thể trên các đối tượng trong cơ sở dữ liệu (như bảng, view, thủ tục lưu trữ...).

## Cấp quyền đọc và ghi (SELECT, INSERT, UPDATE, DELETE) trên một bảng cụ thể:
```
GRANT SELECT, INSERT, UPDATE, DELETE ON [SchemaName].[TableName] TO [Username];
```

+ **SchemaName**: Tên schema của bảng (thường là dbo).
+ **TableName**: Tên bảng bạn muốn cấp quyền.

## Cấp tất cả các quyền trên một schema:
```
GRANT ALL ON SCHEMA::[SchemaName] TO [Username];
```

Thao tác này sẽ cấp quyền đầy đủ trên tất cả các đối tượng (bảng, view, function...) trong schema đó.

## Cấp vai trò (Role) có sẵn:

Bạn có thể thêm User vào các vai trò có sẵn của SQL Server để cấp một bộ quyền định sẵn.

### Vai trò trong cơ sở dữ liệu (Database Roles):
+ **db_datareader**: Chỉ đọc dữ liệu.
+ **db_datawriter**: Chỉ ghi dữ liệu.
+ **db_owner**: Toàn quyền trên cơ sở dữ liệu.

```
-- Cấp quyền đọc và ghi
ALTER ROLE db_datareader ADD MEMBER [Username];
ALTER ROLE db_datawriter ADD MEMBER [Username];

-- Hoặc cấp quyền chủ sở hữu (toàn quyền)
ALTER ROLE db_owner ADD MEMBER [Username];
```

### Vai trò trên Server (Server Roles):
+ **sysadmin**: Toàn quyền trên toàn bộ SQL Server.

```
ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\Username];
```

# Lưu ý quan trọng
+ **Sử dụng tên miền**: Luôn sử dụng cú pháp DOMAIN\Username hoặc DOMAIN\GroupName để chỉ định tài khoản Windows.
+ **Nguyên tắc đặc quyền tối thiểu**: Chỉ cấp những quyền cần thiết để User thực hiện công việc, tránh cấp quyền quá rộng như sysadmin hoặc db_owner nếu không cần thiết.
+ **Sử dụng nhóm**: Thay vì cấp quyền cho từng tài khoản người dùng, hãy tạo các nhóm Windows (ví dụ: "IT Department", "Sales Team") và cấp quyền cho các nhóm này. Sau đó, chỉ cần thêm người dùng vào nhóm tương ứng trên Windows, việc quản lý sẽ trở nên dễ dàng và hiệu quả hơn.

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
