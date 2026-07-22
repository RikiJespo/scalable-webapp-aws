# Scalable Web Application on AWS

Proyek latihan membangun arsitektur web application 3-tier yang scalable di AWS, menggunakan VPC custom, Auto Scaling, Load Balancer, RDS, serta static asset delivery via S3 + CloudFront.

## Arsitektur

![Architecture Diagram](docs/screenshots/21-architecture-diagram.png)

```
Internet
   |
CloudFront (CDN)
   |--- Static assets ---> S3 (private, via OAC)
   |--- Dynamic requests ---> ALB
                                |
                        Auto Scaling Group
                        (EC2 x2, multi-AZ)
                                |
                        RDS MySQL (private, Single-AZ)
```

**Region:** Asia Pacific (Singapore) — `ap-southeast-1`

## Komponen

| Komponen | Deskripsi |
|---|---|
| **VPC** | Custom VPC dengan 2 AZ, public & private subnet |
| **Security Groups** | `alb-sg`, `ec2-sg`, `rds-sg` — layered security per tier |
| **Auto Scaling Group** | `webapp-asg`, multi-AZ, EC2 di private/public subnet |
| **Launch Template** | `webapp-launch-template` — auto install httpd + mariadb via User Data |
| **Application Load Balancer** | Distribusi traffic HTTP ke instance ASG |
| **RDS MySQL** | `riki-webapp-db`, instance `db.t4g.micro`, private, Single-AZ |
| **S3** | `riki-webapp-static-assets` — private bucket untuk static files |
| **CloudFront** | CDN dengan Origin Access Control (OAC) ke S3 |

## Detail Resource

### Networking
- VPC: `riki-latihan-vpc-vpc` (`vpc-0d5cbc645601b49fb`)
- Subnet public: `ap-southeast-1a` & `ap-southeast-1b`
- Subnet private: `ap-southeast-1a` & `ap-southeast-1b`

### Security Groups
- `rds-sg` — Inbound MySQL/3306 dari `ec2-sg`
- `ec2-sg` — Inbound SSH/22 dari IP tertentu, HTTP/80 dari `alb-sg`
- `alb-sg` — Inbound HTTP/80 dari publik (0.0.0.0/0)

### Database
- Identifier: `riki-webapp-db`
- Engine: MySQL 8.4.9, `db.t4g.micro`
- Database: `webapp_db`
- Storage: 20 GiB gp2 (autoscaling dimatikan)
- Akses: Private (tidak public)

## Testing & Verifikasi

| Test | Hasil |
|---|---|
| ALB → ASG (Load Balancing) | ✅ Hostname bergantian antar instance |
| End-to-end (Browser → ALB → EC2) | ✅ Response `Hello from ip-10-0-xx-xxx...` |
| CloudFront → S3 (via OAC) | ✅ Static asset tampil, bucket tetap private |

## Dokumentasi Screenshot

Seluruh proses build & testing didokumentasikan di folder [`docs/screenshots/`](docs/screenshots/):

1. `06-rds-sg-inbound-rules-final.png`
2. `08-mysql-connected-to-rds.png`
3. `10-launch-template-created.png`
4. `11-asg-created.png`
5. `12-asg-instances-running.png`
6. `14-alb-created-active.png`
7. `15-asg-attached-to-tg.png`
8. `16-alb-test-powershell-200ok.png`
9. `16b-alb-test-browser-200ok.png`
10. `17-s3-bucket-static-assets-created.png`
11. `18-cloudfront-distribution-enabled.png`
12. `19-test-cloudfront-success.png`
13. `21-architecture-diagram.svg`

## Isu & Catatan Teknis

- **Browser vs PowerShell (HTTP redirect):** Sempat terjadi timeout saat akses ALB via browser padahal PowerShell curl sukses. Setelah dicoba ulang dengan ALB baru, browser berhasil akses normal.
- **Region:** Semua resource berada di `ap-southeast-1` (Singapore), bukan default region N. Virginia.
- **Public IP EC2:** IP publik instance EC2 standalone berubah setiap kali di-stop/start; tidak berlaku untuk instance dalam ASG karena selalu di-provision baru.
- **RDS Storage Autoscaling:** Dimatikan secara sengaja untuk mencegah biaya storage membengkak otomatis.

## Cleanup / Cost Management

Untuk menghindari biaya idle, resource dimatikan dengan urutan:
1. **CloudFront** → Disable
2. **ALB** → Delete (komponen paling boros jika dibiarkan idle, ~$16-25/bulan)
3. **ASG** → Desired/Min capacity = 0 (instance otomatis terminate)
4. **RDS** → Stop temporarily
5. **S3** → Dibiarkan aktif (biaya storage sangat kecil)

## Tech Stack

- **Compute:** EC2, Auto Scaling Group, Launch Template
- **Networking:** VPC, Subnet, Route Table, Internet Gateway, Security Groups
- **Load Balancing:** Application Load Balancer
- **Database:** RDS (MySQL)
- **Storage & CDN:** S3, CloudFront (OAC)

---

*Dibangun sebagai project latihan arsitektur cloud scalable di AWS.*
