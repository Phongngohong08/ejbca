# Hướng Dẫn Chi Tiết Triển Khai EJBCA CE 9.1.1 Từ Mã Nguồn

## 📋 Tổng Quan Hệ Thống

*   **Hệ điều hành:** Ubuntu 24.04
*   **Application Server:** WildFly Full 32.0.0.Final
*   **Database:** MariaDB 10.11.13
*   **EJBCA Version:** EJBCA CE 9.1.1
*   **Java Version:** OpenJDK 21
*   **Super Admin Password (Mặc định):** `ejbca`

---

## 🔧 Bước 1: Chuẩn Bị Hệ Thống và Cài Đặt Dependencies

### 1.1. Dọn dẹp và Cập nhật hệ thống (Khuyến nghị)

```bash
sudo apt-get update && sudo apt-get autoremove -y && sudo apt-get autoclean -y && sudo apt-get clean && sudo journalctl --vacuum-time=3d && rm -rf ~/.cache/thumbnails/*
```

### 1.2. Cài đặt Java (JDK 21), Ant và Git

```bash
sudo apt install -y openjdk-21-jdk ant git unzip wget
```

### 1.3. Cài đặt và Cấu hình MariaDB

```bash
sudo apt install -y mariadb-server mariadb-client
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

**Tạo Database và User**:

```bash
sudo mysql -u root << EOF
CREATE DATABASE ejbca CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'ejbca'@'localhost' IDENTIFIED BY 'ejbca';
GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca'@'localhost';
FLUSH PRIVILEGES;
EXIT;
EOF
```

---

## 🚀 Bước 2: Cài Đặt và Chuẩn Bị WildFly

### 2.1. Download và Giải nén WildFly

```bash
cd /tmp
wget https://github.com/wildfly/wildfly/releases/download/32.0.0.Final/wildfly-32.0.0.Final.zip -O /tmp/wildfly-32.0.0.Final.zip
sudo unzip -q /tmp/wildfly-32.0.0.Final.zip -d /opt/
sudo ln -sfn /opt/wildfly-32.0.0.Final /opt/wildfly
```

### 2.2. Cấu hình Thư mục và Quyền truy cập

```bash
sudo mkdir -p /opt/wildfly/standalone/{log,data,tmp}
sudo mkdir -p /opt/wildfly/standalone/configuration/keystore
sudo chown -R $USER:$USER /opt/wildfly
```

### 2.3. Set Biến Môi Trường `APPSRV_HOME`

```bash
echo 'export APPSRV_HOME=/opt/wildfly' >> ~/.bashrc
source ~/.bashrc
```

---

## 💾 Bước 3: Deploy MariaDB JDBC Driver & Cấu Hình Datasource

### 3.1. Deploy JDBC Driver

```bash
wget https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/3.1.4/mariadb-java-client-3.1.4.jar -O /opt/wildfly/standalone/deployments/mariadb-java-client.jar
```

### 3.2. Khởi động WildFly

```bash
/opt/wildfly/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &
sleep 20
```

### 3.3. Tạo Credential Store và Alias

```bash
/opt/wildfly/bin/jboss-cli.sh --connect << 'EOF'
/subsystem=elytron/credential-store=defaultCS:add(location="credentials/defaultCS.jceks",relative-to=jboss.server.data.dir,credential-reference={clear-text=changeit},create=true)
/subsystem=elytron/credential-store=defaultCS:add-alias(alias=dbPassword,secret-value="ejbca")
EOF
```

### 3.4. Thêm Datasource `ejbcaDS`

```bash
/opt/wildfly/bin/jboss-cli.sh --connect << 'EOF'
data-source add --name=ejbcaDS --jndi-name="java:/EjbcaDS" --driver-name="mariadb-java-client.jar" --driver-class="org.mariadb.jdbc.Driver" --connection-url="jdbc:mariadb://127.0.0.1:3306/ejbca" --user-name="ejbca" --credential-reference={store=defaultCS,alias=dbPassword} --min-pool-size=5 --max-pool-size=50 --transaction-isolation=TRANSACTION_READ_COMMITTED
EOF
```

---

## 📦 Bước 4: Build và Deploy EJBCA

### 4.1. Clone EJBCA Source Code

```bash
cd ~
git clone https://github.com/Keyfactor/ejbca-ce.git
cd ejbca-ce
```

### 4.2. Cấu hình Properties

```bash
cp conf/database.properties.sample conf/database.properties
cp conf/ejbca.properties.sample conf/ejbca.properties
cp conf/web.properties.sample conf/web.properties

sed -i 's/^#database.name=mysql/database.name=mysql/' conf/database.properties
sed -i 's|^#database.url=jdbc:h2.*|database.url=jdbc:mariadb://127.0.0.1:3306/ejbca|' conf/database.properties
sed -i 's/^#database.driver=.*/database.driver=org.mariadb.jdbc.Driver/' conf/database.properties
```

### 4.3. Build và Deploy

```bash
sudo chown -R $USER:$USER ~/ejbca-ce
cd ~/ejbca-ce
ant clean build
ant deployear
```

---

## 🔧 Bước 5: Cấu Hình EJB Client Port & Khởi Tạo EJBCA

### 5.1. Cấu Hình EJB Client Port

```bash
cd ~/ejbca-ce
sed -i 's/^remote.connection.default.port = 4447$/remote.connection.default.port = 8080/' ~/ejbca-ce/dist/ejbca-ejb-cli/jboss-ejb-client.properties
```

### 5.2. Khởi Tạo EJBCA

```bash
cd ~/ejbca-ce
ant runinstall
```

---

## 🔐 Bước 6: Cấu Hình TLS và SSL Client Authentication

### 6.1. Deploy TLS Keystores

```bash
cd ~/ejbca-ce
ant deploy-keystore
```

### 6.2. Restart WildFly và Cấu hình SSL Client Auth

```bash
pkill -9 -f wildfly
sleep 3

/opt/wildfly/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &
sleep 30

cat << 'EOF' > /tmp/configure-ssl.cli
/subsystem=elytron/key-store=trustKS:add(path=keystore/truststore.p12,relative-to=jboss.server.config.dir,credential-reference={clear-text=changeit},type=PKCS12)
/subsystem=elytron/trust-manager=trustManager:add(key-store=trustKS)
/subsystem=elytron/server-ssl-context=applicationSSC:write-attribute(name=need-client-auth,value=true)
/subsystem=elytron/server-ssl-context=applicationSSC:write-attribute(name=trust-manager,value=trustManager)
:reload
EOF

/opt/wildfly/bin/jboss-cli.sh --connect --file=/tmp/configure-ssl.cli
sleep 20
```

---

## 🎉 Bước 7: Truy Cập EJBCA Admin Web

### 7.1. Copy Super Admin Certificate

```bash
cp ~/ejbca-ce/p12/superadmin.p12 ~/Documents/
ls -lh ~/Documents/superadmin.p12
```

### 7.2. Import Certificate vào Browser

Password: `ejbca`

---

## 🔍 Kiểm tra và Xác minh Cuối cùng

```bash
ps aux | grep wildfly | grep -v grep
ss -tlnp | grep -E ":(8080|8443|9990)"
/opt/wildfly/bin/jboss-cli.sh --connect --command="/subsystem=elytron/server-ssl-context=applicationSSC:read-resource()" 2>&1 | grep need-client-auth
curl -k -s -o /dev/null -w "HTTP Code: %{http_code}\n" https://localhost:8443/ejbca/adminweb/
curl --cert ~/Documents/superadmin.p12:ejbca --cert-type P12 -k -s -o /dev/null -w "With cert: %{http_code}\n" https://localhost:8443/ejbca/adminweb/
```

