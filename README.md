# Zabbix Proxy 8.0 dengan Docker Compose

Deploy Active Proxy untuk Zabbix Server 8.0.x menggunakan image resmi `zabbix/zabbix-proxy-mysql` dan MySQL 8.4. Proxy mengirim data ke Zabbix Server melalui TCP 10051. MySQL menyimpan data di volume `mysql-data`.

## Prasyarat

- Zabbix Server 8.0.x yang dapat dijangkau dari host Docker melalui TCP 10051.
- Docker Engine 28.0 atau lebih baru dengan Docker Compose v2 untuk bridge dengan gateway mode `isolated`.
- Akses frontend Zabbix untuk mendaftarkan proxy dan menetapkan host yang dipantau.

## Instalasi

1. Buat `.env` dengan izin baca dan tulis untuk pemilik file. Jalankan `openssl` dua kali untuk menghasilkan password yang berbeda.

   ```bash
   install -m 600 .env.example .env
   openssl rand -hex 32
   openssl rand -hex 32
   ```

2. Isi `ZABBIX_SERVER_HOST` dengan IP atau DNS Zabbix Server, lalu masukkan password untuk `MYSQL_PASSWORD` dan `MYSQL_ROOT_PASSWORD`. Gunakan format `host:port` jika Zabbix Server memakai port selain 10051. Untuk deployment ulang, gunakan `.env` yang sudah ada. Password MySQL ditetapkan saat volume pertama kali diinisialisasi.
3. Di frontend, buka **Administration → Proxies → Create proxy**. Isi **Proxy name** sesuai `ZABBIX_PROXY_NAME`, pilih **Active**, dan isi **Proxy address** dengan IP sumber yang terlihat oleh Zabbix Server. Konfigurasi dasar memakai **No encryption** pada **Connections from proxy**. Untuk koneksi lintas lokasi, gunakan TLS PSK atau sertifikat di kedua sisi.
4. Validasi dan jalankan Compose.

   ```bash
   docker compose config --quiet
   docker compose pull
   docker compose up -d
   docker compose ps
   docker compose logs --tail=100 zabbix-proxy
   ```

## Jaringan dan firewall

| Network | Anggota | Fungsi |
| --- | --- | --- |
| `db_private` | `mysql`, `zabbix-proxy` | Bridge internal dengan gateway IPv4 `isolated`. Proxy mengakses database melalui `mysql:3306`. Port MySQL berada di jaringan Docker. |
| `monitoring` | `zabbix-proxy` | Bridge untuk koneksi ke Zabbix Server dan perangkat yang dipantau. |

| Sumber | Tujuan | Port | Kebutuhan |
| --- | --- | --- | --- |
| Proxy | Zabbix Server | TCP 10051 | Active Proxy |
| Proxy | Zabbix Agent atau Agent 2 | TCP 10050 | Passive agent checks |
| Zabbix Agent atau Agent 2 | Host Docker proxy | TCP 10051 | Active agent checks |
| Proxy | Perangkat SNMP | UDP 161 | SNMP polling |

Untuk menerima active agent checks, buat `compose.override.yaml` pada host proxy. Ganti `192.0.2.20` dengan IP internal host Docker yang dapat dijangkau agent.

```yaml
services:
  zabbix-proxy:
    ports:
      - "192.0.2.20:10051:10051"
```

Jalankan kembali `docker compose up -d`. Atur `ServerActive=<IP-host-Docker>:10051` pada agent. Untuk passive agent checks, isi `Server=` pada agent dengan IP sumber proxy yang terlihat oleh agent.

## Verifikasi

1. Jalankan `docker compose ps`. Pastikan `mysql` berstatus `healthy`, `zabbix-proxy` berjalan, dan MySQL tidak memiliki pemetaan port ke host.
2. Buka **Administration → Proxies**. Pastikan versi proxy `8.0.x` dan **Last seen (age)** terus diperbarui.
3. Buka **Data collection → Hosts**. Atur satu host uji menjadi **Monitored by → Proxy**, pilih proxy Anda, lalu periksa pembaruan metrik di **Monitoring → Latest data**.

## Referensi

- [Image resmi Zabbix Proxy MySQL](https://hub.docker.com/r/zabbix/zabbix-proxy-mysql)
- [Dokumentasi Zabbix 8.0 untuk distributed monitoring](https://www.zabbix.com/documentation/8.0/en/manual/concepts/proxy/distributed_monitoring)
- [Jaringan Docker Compose](https://docs.docker.com/compose/how-tos/networking/)
- [Gateway mode bridge Docker](https://docs.docker.com/engine/network/port-publishing/#gateway-modes)
