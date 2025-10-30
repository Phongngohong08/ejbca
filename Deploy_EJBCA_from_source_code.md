# Hướng Dẫn Cài Đặt EJBCA CE 9.1.1 trên Ubuntu 24.04

## 📋 Tổng Quan

Tài liệu này hướng dẫn cài đặt và cấu hình EJBCA Community Edition 9.1.1 trên Ubuntu 24.04 với:
- **WildFly 32.0.0.Final** - Application Server
- **MariaDB 10.11.13** - Database
- **OpenJDK 21** - Java Runtime
- **Apache Ant 1.10.14** - Build Tool

---

## 🔧 Bước 1: Cài Đặt Dependencies

### 1.1. Cài đặt Java, Ant và Git
```bash
sudo apt update
sudo apt install -y openjdk-21-jdk ant git
```

### 1.2. Cài đặt MariaDB
```bash
sudo apt install -y mariadb-server mariadb-client
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

### 1.3. Tạo Database và User
```bash
mysql -u root -e "CREATE DATABASE ejbca CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -e "CREATE USER 'ejbca'@'localhost' IDENTIFIED BY 'ejbca';"
mysql -u root -e "GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca'@'localhost';"
mysql -u root -e "FLUSH PRIVILEGES;"
```

---

## 🚀 Bước 2: Cài Đặt WildFly

### 2.1. Download và Giải Nén WildFly
```bash
cd /tmp
wget https://github.com/wildfly/wildfly/releases/download/32.0.0.Final/wildfly-32.0.0.Final.tar.gz
sudo tar -xzf wildfly-32.0.0.Final.tar.gz -C /opt/
sudo mv /opt/wildfly-32.0.0.Final /opt/wildfly
```

### 2.2. Tạo Thư Mục Cần Thiết
```bash
sudo mkdir -p /opt/wildfly/standalone/{log,data,tmp}
sudo mkdir -p /opt/wildfly/standalone/configuration/keystore
sudo chown -R $USER:$USER /opt/wildfly
sudo chown -R $USER:$USER /opt/wildfly-32.0.0.Final
```

### 2.3. Set Biến Môi Trường
```bash
echo 'export APPSRV_HOME=/opt/wildfly' >> ~/.bashrc
source ~/.bashrc
```

---

## 💾 Bước 3: Deploy MariaDB JDBC Driver

### 3.1. Download JDBC Driver
```bash
cd /tmp
wget https://dlm.mariadb.com/3906418/Connectors/java/connector-java-3.1.4/mariadb-java-client-3.1.4.jar
sudo cp mariadb-java-client-3.1.4.jar /opt/wildfly/standalone/deployments/
```

### 3.2. Khởi Động WildFly Lần Đầu
```bash
/opt/wildfly/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &
sleep 20  # Đợi WildFly khởi động
```

---

## 🔐 Bước 4: Cấu Hình Datasource

### 4.1. Tạo Credential Store
```bash
/opt/wildfly/bin/jboss-cli.sh --connect << 'EOF'
/subsystem=elytron/credential-store=ejbcaCredStore:add(
    location=/opt/wildfly/standalone/configuration/ejbca-credentials.store,
    create=true,
    credential-reference={clear-text=storepassword}
)
/subsystem=elytron/credential-store=ejbcaCredStore:add-alias(
    alias=dbPassword,
    secret-value=ejbca
)
EOF
```

### 4.2. Thêm Datasource
```bash
/opt/wildfly/bin/jboss-cli.sh --connect << 'EOF'
data-source add \
    --name=ejbcaDS \
    --jndi-name=java:/EjbcaDS \
    --driver-name=mariadb-java-client-3.1.4.jar \
    --connection-url=jdbc:mariadb://127.0.0.1:3306/ejbca \
    --user-name=ejbca \
    --credential-reference={store=ejbcaCredStore,alias=dbPassword} \
    --min-pool-size=5 \
    --max-pool-size=50 \
    --pool-prefill=true \
    --transaction-isolation=TRANSACTION_READ_COMMITTED \
    --enabled=true
EOF
```

### 4.3. Test Connection
```bash
/opt/wildfly/bin/jboss-cli.sh --connect \
    --command="/subsystem=datasources/data-source=ejbcaDS:test-connection-in-pool"
```

**Kết quả mong đợi:** `"outcome" => "success"`

---

## 📦 Bước 5: Build và Deploy EJBCA

### 5.1. Clone EJBCA Source Code
```bash
cd ~
git clone https://github.com/Keyfactor/ejbca-ce.git
cd ejbca-ce
sudo chown -R $USER:$USER .
```

### 5.2. Cấu Hình Database
```bash
# Edit database.properties
sed -i 's/^#database.name=mysql/database.name=mysql/' ~/ejbca-ce/conf/database.properties
sed -i 's|^#database.url=.*|database.url=jdbc:mariadb://127.0.0.1:3306/ejbca|' ~/ejbca-ce/conf/database.properties
sed -i 's/^#database.driver=.*/database.driver=org.mariadb.jdbc.Driver/' ~/ejbca-ce/conf/database.properties
```

### 5.3. Build EJBCA
```bash
cd ~/ejbca-ce
ant clean build
```

**Thời gian:** ~40-45 giây

### 5.4. Deploy EJBCA
```bash
ant deployear
```

### 5.5. Đợi Deployment Hoàn Tất
```bash
sleep 30
ls -lh /opt/wildfly/standalone/deployments/ejbca.ear*
```

**Kiểm tra:** File `ejbca.ear.deployed` phải xuất hiện

---

## 🔧 Bước 6: Cấu Hình EJB Client

### 6.1. Update Port cho EJB Remoting
```bash
sed -i 's/^remote.connection.default.port = 4447$/remote.connection.default.port = 8080/' \
    ~/ejbca-ce/dist/ejbca-ejb-cli/jboss-ejb-client.properties
```

**Lý do:** WildFly 32 sử dụng HTTP remoting trên port 8080 thay vì port 4447

---

## 🏗️ Bước 7: Khởi Tạo EJBCA (runinstall)

### 7.1. Chạy Installation
```bash
cd ~/ejbca-ce
ant runinstall
```

**Các giá trị mặc định:**
- CA name: `ManagementCA`
- CA DN: `CN=ManagementCA,O=EJBCA Sample,C=SE`
- Key type: `RSA`
- Key spec: `2048`
- Signature algorithm: `SHA256WithRSA`
- Validity: `3650` days (10 năm)
- CA token password: Nhấn Enter (sẽ tự sinh)

**Kết quả:**
- ✅ Management CA được tạo
- ✅ Database schema được khởi tạo (70+ tables)
- ✅ Super Admin certificate: `~/ejbca-ce/p12/superadmin.p12`
- ✅ Server certificate: `~/ejbca-ce/p12/tomcat.p12`
- ✅ Truststore: `~/ejbca-ce/p12/truststore.p12`

---

## 🔐 Bước 8: Deploy TLS Keystores

### 8.1. Deploy Keystores
```bash
cd ~/ejbca-ce
ant deploy-keystore
```

### 8.2. Restart WildFly
```bash
pkill -9 -f wildfly
sleep 3
/opt/wildfly/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &
sleep 30  # Đợi WildFly khởi động
```

### 8.3. Verify Keystores
```bash
ls -lh /opt/wildfly/standalone/configuration/keystore/
```

**Phải có 2 files:**
- `keystore.p12` - Server TLS certificate
- `truststore.p12` - CA certificates

---

## 🔐 Bước 8.4: Cấu Hình SSL Client Authentication

**QUAN TRỌNG:** Để Admin Web yêu cầu client certificate, cần cấu hình WildFly SSL context.

### 8.4.1. Tạo CLI Script
```bash
cat << 'EOF' > /tmp/configure-ssl.cli
# Create trust-manager using the truststore
/subsystem=elytron/key-store=trustKS:add(path=keystore/truststore.p12,relative-to=jboss.server.config.dir,credential-reference={clear-text=changeit},type=PKCS12)
/subsystem=elytron/trust-manager=trustManager:add(key-store=trustKS)

# Update server-ssl-context to require client authentication
/subsystem=elytron/server-ssl-context=applicationSSC:write-attribute(name=need-client-auth,value=true)
/subsystem=elytron/server-ssl-context=applicationSSC:write-attribute(name=trust-manager,value=trustManager)

# Reload server
:reload
EOF
```

### 8.4.2. Chạy CLI Script
```bash
/opt/wildfly/bin/jboss-cli.sh --connect --file=/tmp/configure-ssl.cli
```

**Kết quả mong đợi:**
```
{"outcome" => "success"}
{"outcome" => "success"}
{"outcome" => "success", ...}
{"outcome" => "success", ...}
{"outcome" => "success"}
```

### 8.4.3. Đợi WildFly Reload
```bash
sleep 20
```

### 8.4.4. Verify SSL Configuration
```bash
# Test WITHOUT certificate (should fail)
curl -k -s -o /dev/null -w "Without cert: %{http_code}\n" https://localhost:8443/ejbca/adminweb/

# Test WITH certificate (should succeed)
curl --cert ~/Documents/superadmin.p12:ejbca --cert-type P12 -k -s -o /dev/null -w "With cert: %{http_code}\n" https://localhost:8443/ejbca/adminweb/
```

**Kết quả mong đợi:**
```
Without cert: 000  (SSL handshake failed - GOOD!)
With cert: 200     (Access granted - GOOD!)
```

### 8.4.5. Verify CLI Configuration
```bash
/opt/wildfly/bin/jboss-cli.sh --connect --command="/subsystem=elytron/server-ssl-context=applicationSSC:read-resource()" | grep -E "(need-client-auth|trust-manager)"
```

**Kết quả mong đợi:**
```
"need-client-auth" => true,
"trust-manager" => "trustManager",
```

---

## 🎉 Bước 9: Truy Cập EJBCA

### 9.1. Copy Super Admin Certificate
```bash
cp ~/ejbca-ce/p12/superadmin.p12 ~/Documents/
```

### 9.2. Import Certificate vào Browser

#### **Firefox:**
1. Mở `Settings` → `Privacy & Security`
2. Cuộn xuống `Certificates` → Click `View Certificates`
3. Tab `Your Certificates` → Click `Import`
4. Chọn file `~/Documents/superadmin.p12`
5. Nhập password: **`ejbca`**
6. Click **OK**

#### **Chrome/Chromium:**
1. Mở `Settings` → `Privacy and Security` → `Security`
2. Cuộn xuống `Manage certificates`
3. Tab `Your certificates` → Click `Import`
4. Chọn file `~/Documents/superadmin.p12`
5. Nhập password: **`ejbca`**
6. Click **OK**

**⚠️ QUAN TRỌNG:** Sau khi import certificate, **đóng HOÀN TOÀN browser** (không chỉ đóng tab) và mở lại.

### 9.3. Truy Cập Admin Web

1. Mở browser và truy cập: **https://localhost:8443/ejbca/adminweb/**
2. Browser sẽ hiển thị **Certificate Warning** (do self-signed):
   - Firefox: Click **"Advanced"** → **"Accept the Risk and Continue"**
   - Chrome: Click **"Advanced"** → **"Proceed to localhost (unsafe)"**
3. Browser sẽ hiển thị **"User Identification Request"** (chọn certificate):
   - Chọn certificate **`SuperAdmin`** hoặc **`CN=SuperAdmin`**
   - Click **OK** hoặc **Allow**
4. Bạn sẽ thấy EJBCA Admin Web interface

**Nếu không thấy prompt chọn certificate:**
- Đảm bảo đã đóng hoàn toàn browser và mở lại
- Kiểm tra certificate đã import: Settings → Certificates → Your Certificates
- Xóa certificate cũ nếu có và import lại

### 9.4. Tổng Hợp Các URLs

| Interface | URL | Yêu Cầu |
|-----------|-----|---------|
| **RA Web** | http://localhost:8080/ejbca/ra/ | Không cần cert |
| **Admin Web** | https://localhost:8443/ejbca/adminweb/ | Cần superadmin.p12 |
| **REST API** | http://localhost:8080/ejbca/ejbca-rest-api/ | Tùy API |
| **Public Web** | http://localhost:8080/ejbca/ | Không cần cert |

---

## 🔍 Kiểm Tra và Xác Minh

### Kiểm tra WildFly
```bash
ps aux | grep wildfly | grep -v grep
```

### Kiểm tra Database Tables
```bash
mysql -u ejbca -pejbca ejbca -e "SHOW TABLES;" | wc -l
```
**Kết quả:** Phải có 70+ tables

### Kiểm tra HTTP/HTTPS
```bash
# Test HTTP
curl -s http://localhost:8080/ejbca/ra/ -o /dev/null -w "HTTP: %{http_code}\n"

# Test HTTPS
curl -k -s https://localhost:8443/ejbca/adminweb/ -o /dev/null -w "HTTPS: %{http_code}\n"
```
**Kết quả mong đợi:** Cả 2 đều trả về `200`

### Kiểm tra Datasource
```bash
/opt/wildfly/bin/jboss-cli.sh --connect \
    --command="/subsystem=datasources/data-source=ejbcaDS:test-connection-in-pool"
```

---

## 🛠️ Quản Lý và Vận Hành

### Khởi Động WildFly
```bash
/opt/wildfly/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &
```

### Dừng WildFly
```bash
pkill -9 -f wildfly
```

### Xem Log
```bash
tail -f /opt/wildfly/standalone/log/server.log
```

### Sử Dụng EJBCA CLI
```bash
cd ~/ejbca-ce/dist/ejbca-ejb-cli
java -jar ejbca-ejb-cli.jar ca listcas
java -jar ejbca-ejb-cli.jar ca info ManagementCA
```

---

## 🔄 Reset và Cài Đặt Lại

### Reset Hoàn Toàn
```bash
# 1. Dừng WildFly
pkill -9 -f wildfly

# 2. Xóa Database
mysql -u ejbca -pejbca -e "DROP DATABASE IF EXISTS ejbca; CREATE DATABASE ejbca CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# 3. Xóa Certificates
rm -rf ~/ejbca-ce/p12/*
rm -f ~/Documents/superadmin.p12

# 4. Xóa Deployment
rm -f /opt/wildfly/standalone/deployments/ejbca.ear*

# 5. Khởi động lại WildFly
/opt/wildfly/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &
sleep 30

# 6. Deploy lại
cd ~/ejbca-ce
ant deployear
sleep 30

# 7. Chạy install
ant runinstall

# 8. Deploy keystores
ant deploy-keystore

# 9. Restart WildFly
pkill -9 -f wildfly
sleep 3
/opt/wildfly/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &
sleep 30

# 10. Configure SSL Client Authentication
/opt/wildfly/bin/jboss-cli.sh --connect --file=/tmp/configure-ssl.cli
sleep 20

# 11. Copy certificate
cp ~/ejbca-ce/p12/superadmin.p12 ~/Documents/
```

---

## 📝 Thông Tin Quan Trọng

### Credentials
| Thành phần | Username | Password |
|-----------|----------|----------|
| MariaDB | `ejbca` | `ejbca` |
| Super Admin P12 | - | `ejbca` |
| Credential Store | - | `storepassword` |

### Ports
| Service | Port | Protocol |
|---------|------|----------|
| HTTP | 8080 | HTTP |
| HTTPS | 8443 | HTTPS |
| Management | 9990 | HTTP |

### Đường Dẫn Files
| File/Directory | Path |
|----------------|------|
| WildFly Home | `/opt/wildfly` |
| EJBCA Source | `~/ejbca-ce` |
| Keystores | `/opt/wildfly/standalone/configuration/keystore/` |
| Certificates | `~/ejbca-ce/p12/` |
| Super Admin Cert | `~/Documents/superadmin.p12` |
| Server Log | `/opt/wildfly/standalone/log/server.log` |

### Database Schema
- **Total Tables:** 70+
- **Key Tables:**
  - `CAData` - CA information
  - `CertificateData` - Issued certificates
  - `UserData` - End entities
  - `AccessRulesData` - Authorization rules
  - `AdminGroupData` - Admin groups
  - `AuditRecordData` - Audit logs

---

## ⚠️ Troubleshooting

### WildFly không khởi động
```bash
# Kiểm tra log
tail -50 /opt/wildfly/standalone/log/server.log

# Kiểm tra port đã được sử dụng chưa
ss -tlnp | grep -E ":(8080|8443|9990)"
```

### EJBCA CLI không kết nối được
```bash
# Kiểm tra port 8080 có listening không
ss -tlnp | grep 8080

# Verify EJB client config
cat ~/ejbca-ce/dist/ejbca-ejb-cli/jboss-ejb-client.properties
# Port phải là 8080, không phải 4447
```

### Datasource connection failed
```bash
# Test connection
/opt/wildfly/bin/jboss-cli.sh --connect \
    --command="/subsystem=datasources/data-source=ejbcaDS:test-connection-in-pool"

# Kiểm tra MariaDB
sudo systemctl status mariadb
mysql -u ejbca -pejbca ejbca -e "SELECT 1;"
```

### HTTPS không hoạt động
```bash
# Kiểm tra keystores đã deploy chưa
ls -lh /opt/wildfly/standalone/configuration/keystore/

# Nếu chưa có, deploy lại
cd ~/ejbca-ce
ant deploy-keystore

# Restart WildFly
pkill -9 -f wildfly
/opt/wildfly/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &
```

### Browser không hiển thị prompt chọn certificate

**Triệu chứng:** Khi truy cập Admin Web, không thấy dialog yêu cầu chọn certificate, chỉ thấy lỗi "Authorization Denied - No client certificate was presented"

**Nguyên nhân:** WildFly chưa được cấu hình để yêu cầu client certificate (need-client-auth)

**Giải pháp:**

1. **Kiểm tra cấu hình SSL hiện tại:**
```bash
/opt/wildfly/bin/jboss-cli.sh --connect \
  --command="/subsystem=elytron/server-ssl-context=applicationSSC:read-resource()" \
  | grep need-client-auth
```

Nếu kết quả là `"need-client-auth" => false` hoặc `undefined`, cần cấu hình lại.

2. **Cấu hình SSL Client Authentication:**
```bash
# Tạo CLI script
cat << 'EOF' > /tmp/configure-ssl.cli
/subsystem=elytron/key-store=trustKS:add(path=keystore/truststore.p12,relative-to=jboss.server.config.dir,credential-reference={clear-text=changeit},type=PKCS12)
/subsystem=elytron/trust-manager=trustManager:add(key-store=trustKS)
/subsystem=elytron/server-ssl-context=applicationSSC:write-attribute(name=need-client-auth,value=true)
/subsystem=elytron/server-ssl-context=applicationSSC:write-attribute(name=trust-manager,value=trustManager)
:reload
EOF

# Chạy CLI script
/opt/wildfly/bin/jboss-cli.sh --connect --file=/tmp/configure-ssl.cli

# Đợi reload
sleep 20
```

3. **Verify cấu hình:**
```bash
# Test WITHOUT certificate (should fail with SSL error)
curl -k -s -o /dev/null -w "Without cert: %{http_code}\n" https://localhost:8443/ejbca/adminweb/

# Test WITH certificate (should return 200)
curl --cert ~/Documents/superadmin.p12:ejbca --cert-type P12 -k -s -o /dev/null -w "With cert: %{http_code}\n" https://localhost:8443/ejbca/adminweb/
```

Kết quả mong đợi:
- Without cert: `000` (SSL handshake failed)
- With cert: `200` (Success)

4. **Trong browser:**
   - Đóng HOÀN TOÀN browser (tất cả cửa sổ)
   - Mở lại và truy cập https://localhost:8443/ejbca/adminweb/
   - Giờ browser sẽ hiển thị prompt chọn certificate

### Certificate không được trust trong browser

**Triệu chứng:** Browser hiển thị lỗi SSL/TLS về certificate không tin cậy

**Giải pháp:**

# Nếu chưa có, deploy lại
cd ~/ejbca-ce
ant deploy-keystore

# Restart WildFly
pkill -9 -f wildfly
/opt/wildfly/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &
```

---