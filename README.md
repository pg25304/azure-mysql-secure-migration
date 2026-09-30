# Secure MySQL Migration to Azure

A hands-on cloud migration project demonstrating the secure migration of a local **MySQL 8.4 database** to **Azure Database for MySQL Flexible Server** using **Azure Database Migration Service (DMS)**, private Azure networking, an **IPsec/IKEv2 site-to-site VPN**, and **TLS-encrypted database connections**.

The project went beyond simply transferring a database. I designed and troubleshot the networking, database connectivity, migration permissions, private DNS resolution and security controls required to create a working hybrid migration path between my local lab and Microsoft Azure.

---

## Project Objectives

The main objectives were to:

- Build and prepare a local MySQL database for migration.
- Migrate the database to Azure Database for MySQL Flexible Server.
- Use Azure Database Migration Service for the migration.
- Keep the Azure database on private networking.
- Establish secure connectivity between the local environment and Azure.
- Protect database traffic using TLS.
- Apply least-privilege principles to the migration account.
- Validate the migrated data independently after migration.
- Document troubleshooting and lessons learned from the implementation.

---

## Architecture

The final migration path was:

**Local Kali Linux VM**  
↓  
**Docker – MySQL 8.4**  
↓  
**StrongSwan IPsec/IKEv2 Site-to-Site VPN**  
↓  
**Azure VPN Gateway**  
↓  
**Azure Migration VNet (10.10.0.0/16)**  
↓  
**Azure Database Migration Service**  
↓  
**VNet Peering + Azure Private DNS**  
↓  
**Azure Database for MySQL Flexible Server**

A temporary Ubuntu VM inside the Azure migration VNet was also used to independently test DNS resolution, network connectivity, TLS and the final migrated database.

---

## Technologies Used

| Technology | Purpose |
|---|---|
| Microsoft Azure | Cloud platform |
| Azure Database for MySQL Flexible Server | Migration target |
| Azure Database Migration Service | Offline database migration |
| Azure Virtual Network | Private Azure networking |
| VNet Peering | Connectivity between Azure networks |
| Azure Private DNS | Private MySQL name resolution |
| Azure VPN Gateway | Hybrid network connectivity |
| StrongSwan | Local IPsec/IKEv2 VPN endpoint |
| Docker | MySQL source environment |
| MySQL 8.4 | Source database |
| Kali Linux | Local lab environment |
| Ubuntu | Azure validation VM |
| TLS | Database traffic encryption |

---

## Source Database

The final source environment used the official **MySQL 8.4 Docker image**.

The test database was:

`cloud_migration_lab`

It contained two application tables:

- `customers`
- `orders`

A dedicated `migration_user` account was used for the migration instead of performing the DMS operation using the MySQL root account.

The required migration privileges were added after DMS source validation identified missing permissions.

---

## From MariaDB to MySQL

The project originally started with **MariaDB** on Kali Linux.

During testing, I encountered compatibility and authentication issues with the migration workflow. I therefore reviewed the requirements of the selected Azure DMS migration path and changed the final source platform to **Oracle MySQL 8.4**.

Moving to MySQL was an important design decision rather than simply continuing with an environment that was causing compatibility problems.

Further local authentication testing eventually led me to run MySQL using the official Docker image, giving me a clean and predictable source environment for the migration.

---

## Secure Hybrid Connectivity

The Azure MySQL target was kept on **private networking**.

Rather than exposing the database directly to the Internet, I established an **IPsec/IKEv2 site-to-site VPN** between the local Kali environment and Azure.

StrongSwan provided the local VPN endpoint, while Azure VPN Gateway provided the Azure endpoint.

During implementation, I also encountered a real-world issue caused by changes to the local public IP address. Updating the Azure Local Network Gateway restored the correct VPN endpoint configuration.

The final VPN connection successfully reached **Connected** status.

---

## Network Redesign

One of the most important troubleshooting stages involved the original Azure network architecture.

The original DMS VNet used:

`10.0.0.0/16`

The Azure MySQL VNet used:

`10.0.0.0/24`

These address spaces overlapped, preventing the VNets from being peered.

Rather than altering the existing MySQL network, I created a new migration network:

`mysql-migration-dms-vnet-v2`

Address space:

`10.10.0.0/16`

Gateway subnet:

`10.10.1.0/24`

The new VNet was successfully peered with the MySQL VNet.

The Azure MySQL **Private DNS zone** was then linked to the new migration VNet so that the MySQL server hostname resolved to its private Azure address.

This redesign resolved the target connectivity problem and provided a clean migration path.

---

## TLS Validation

Database connectivity was tested with TLS explicitly required.

Example:

```bash
mysql --connect-timeout=10 \
  --protocol=TCP \
  --ssl-mode=REQUIRED \
  -h <SOURCE_PRIVATE_IP> \
  -P 3307 \
  -u migration_user \
  -p
