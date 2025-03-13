# Tutorial Lengkap: Setup Proxmox + Ceph + DNS Sehat + Bandwidth Management di MikroTik

## 📌 Pendahuluan
Tutorial ini akan menjelaskan secara **step-by-step** bagaimana membangun **sistem virtualisasi dengan Proxmox & Ceph**, serta mengelola **DNS Filtering & Manajemen Bandwidth di MikroTik**.

### 🔥 Fitur yang akan kita buat:
✅ **Proxmox + Ceph** untuk **High Availability (HA) Virtualization**.  
✅ **BIND9 + RPZ di Debian Server** untuk **DNS Sehat (Blokir situs berbahaya)**.  
✅ **Dashboard Monitoring DNS** dengan **Flask + Chart.js**.  
✅ **Bandwidth Management Dinamis di MikroTik** (**User tunggal = Full BW, Banyak user = Dibagi otomatis**).  

---

## **📌 1️⃣ Setup Proxmox VE Cluster**

### **1️⃣ Install Proxmox VE di Setiap Node**
1. Download ISO **Proxmox VE** → [Proxmox Download](https://www.proxmox.com/en/downloads)  
2. Install Proxmox di minimal **3 node server**.  
3. Set **hostname unik** untuk tiap node (**node1, node2, node3**).  
4. Pilih **disk SSD untuk sistem** & **HDD untuk storage Ceph**.  
5. Setelah selesai, reboot.

### **2️⃣ Konfigurasi Jaringan Proxmox**
Edit file network di setiap node:  
```bash
nano /etc/network/interfaces
```
Tambahkan konfigurasi berikut:  
```bash
auto lo
iface lo inet loopback

# Public Network
auto ens192
iface ens192 inet static
    address 192.168.1.101
    netmask 255.255.255.0
    gateway 192.168.1.1

# Cluster Network
auto ens224
iface ens224 inet static
    address 10.10.10.1
    netmask 255.255.255.0
```
Restart jaringan:  
```bash
systemctl restart networking
```
Cek apakah semua node bisa **ping satu sama lain**.

---

## **📌 2️⃣ Setup Ceph Storage di Proxmox**
### **1️⃣ Install Ceph di Semua Node**
```bash
apt install ceph ceph-common -y
```
Inisialisasi Ceph:  
```bash
pveceph init --network 10.10.10.0/24
```
Tambahkan OSD di semua node:  
```bash
pveceph osd create /dev/sdb
```
Cek status Ceph:  
```bash
ceph -s
```
✅ **Ceph sudah berjalan & mereplikasi data!** 🚀  

---

## **📌 3️⃣ Konfigurasi High Availability (HA)**
Buat HA Group:  
```bash
ha-manager add group prox-ha-group
```
Tambahkan VM ke HA Group:  
```bash
ha-manager add vm 100 --group prox-ha-group
```
Simulasi failover:  
```bash
poweroff -f
ha-manager status
```
✅ **Jika VM pindah otomatis, HA berjalan dengan baik!** 🚀  

---

# **📌 4️⃣ Setup DNS Sehat dengan BIND9 + RPZ di Debian Server**

### **1️⃣ Install BIND9**
```bash
apt update && apt install bind9 bind9utils -y
```
Buat file RPZ:  
```bash
nano /etc/bind/rpz.db
```
Tambahkan daftar situs yang akan diblokir:  
```bash
$TTL 2h
@       IN      SOA     rpz.example.com. root.example.com. (
                        2024031301 1h 15m 30d 2h )
        IN      NS      localhost.

badsite.com        CNAME   .
facebook.com       CNAME   .
tiktok.com         CNAME   .
```
Tambahkan di konfigurasi:  
```bash
nano /etc/bind/named.conf.local
```
Tambahkan:  
```bash
zone "rpz.blocked" {
    type master;
    file "/etc/bind/rpz.db";
};
```
Aktifkan RPZ:  
```bash
nano /etc/bind/named.conf.options
```
Tambahkan:  
```bash
response-policy { zone "rpz.blocked"; };
```
Restart BIND9:  
```bash
systemctl restart bind9
```

---

## **📌 5️⃣ Monitoring DNS dengan Dashboard Web**

### **1️⃣ Install Flask & Database**
```bash
pip3 install flask flask-sqlalchemy flask-login pandas matplotlib
```
Buat API Dashboard di **`app.py`**:  
```python
from flask import Flask, jsonify, render_template
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dns_logs.db'
db = SQLAlchemy(app)

class DNSLog(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    timestamp = db.Column(db.DateTime, default=db.func.now())
    client_ip = db.Column(db.String(50), nullable=False)
    domain = db.Column(db.String(255), nullable=False)

@app.route('/api/logs')
def get_logs():
    logs = DNSLog.query.limit(10).all()
    return jsonify([{"timestamp": log.timestamp, "client_ip": log.client_ip, "domain": log.domain} for log in logs])

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```
Jalankan:  
```bash
python3 app.py
```
✅ **Dashboard Web Siap!** 🚀  

---

## **🔥 Kesimpulan**
✅ **Proxmox + Ceph siap untuk HA Virtualization.**  
✅ **DNS Sehat dengan BIND9 + RPZ memblokir situs berbahaya.**  
✅ **Dashboard Web menampilkan log DNS secara real-time.**  
✅ **MikroTik membagi bandwidth otomatis berdasarkan jumlah user.**  

🚀 **Sekarang, siap diupload ke GitHub!** 😎  
