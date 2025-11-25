# 🐘 AWS RDS Proxy Lab — Create MariaDB RDS, Store Secrets, Configure Proxy & Connect via EC2

In this lab, I learned how to **deploy a MariaDB Amazon RDS instance, configure security groups, connect from an EC2 instance, create a Secrets Manager secret, build an RDS Proxy, and test the proxy connection**.

---

## 📋 Lab Overview

**Goal:**

* Create a MariaDB RDS instance
* Connect to the DB from an EC2 instance using SSH + MySQL
* Store credentials securely in **AWS Secrets Manager**
* Create and configure an **RDS Proxy**
* Set up all required security groups
* Test connections through the proxy

**Learning Outcomes:**

* Use AWS Console to deploy MariaDB RDS
* Configure RDS instance classes, storage, and availability
* Connect to MySQL using EC2 terminal
* Create Secrets Manager secrets for database credentials
* Understand and configure proxy security group chaining
* Create RDS Proxy and troubleshoot connection issues

---

## 🛠 Step-by-Step Journey

### **Step 1 — Log Into AWS Console**

Used the provided lab credentials and logged into the AWS Management Console.
Region set to **US East (N. Virginia)**.

Also verified that an EC2 instance named **RDS-connect-instance** was running.

---

### **Step 2 — Create a MariaDB RDS Instance**

Navigated to **RDS → Create Database** and configured:

**RDS Configuration:**

| Setting              | Value                              |
| -------------------- | ---------------------------------- |
| Engine               | MariaDB                            |
| Template             | Dev/Test                           |
| DB Identifier        | `codecloud-app-db`                 |
| Master Username      | `admin`                            |
| Password             | `password89`                       |
| Instance Class       | Burstable (db.t3.small)            |
| Storage Type         | GP2                                |
| Allocated Storage    | 20GB                               |
| Storage Auto-Scaling | Disabled                           |
| Connectivity         | Connect to an EC2 compute resource |
| EC2 Target           | `RDS-connect-instance`             |
| Enhanced Monitoring  | Disabled                           |
| Automated Backups    | Disabled                           |
| Deletion Protection  | Disabled                           |

Clicked **Create Database**.
After several minutes, status changed to:

```
Available
```

---

### **Step 3 — Connect to the EC2 Instance**

From the EC2 console → Connect → SSH instructions.

**Commands:**

```bash
chmod 400 "RDS-key-pair.pem"
ssh -i "RDS-key-pair.pem" ec2-user@<EC2-public-DNS>
```

Once inside the EC2 instance → prepared to connect to MariaDB.

---

### **Step 4 — Connect to MariaDB RDS from EC2**

Retrieved the RDS endpoint from the RDS console.

**Connection command:**

```bash
mysql -h <rds-endpoint> -P 3306 -u admin -p
```

Entered the password:

```
password89
```

Connection successful.

---

### **Step 5 — Create a Secrets Manager Secret**

Navigated to **Secrets Manager → Store a new secret**.

**Configuration:**

| Field              | Value                     |
| ------------------ | ------------------------- |
| Secret Type        | RDS Credentials           |
| Username           | `admin`                   |
| Password           | `password89`              |
| Encryption Key     | Default AWS KMS key       |
| Database           | Selected our RDS instance |
| Secret Name        | `kk-rds-secret`           |
| Automatic Rotation | Disabled                  |

Clicked **Store** to save the secret.

---

### **Step 6 — Create Security Groups for RDS Proxy**

Created a new security group:

| Field       | Value       |
| ----------- | ----------- |
| Name        | `proxy`     |
| Description | Proxy rules |
| VPC         | Default VPC |

**Inbound Rule:**

| Type         | Source                     |
| ------------ | -------------------------- |
| MySQL/Aurora | `EC2-RDS-1` security group |

**Outbound Rule:**

| Type         | Destination                |
| ------------ | -------------------------- |
| MySQL/Aurora | `RDS-EC2-1` security group |

Updated the **RDS security group** (`RDS-EC2-1`) to include:

**Inbound Rule:**

| Type         | Source     |
| ------------ | ---------- |
| MySQL/Aurora | `proxy` SG |

Saved rules.

---

### **Step 7 — Create RDS Proxy**

Navigated to **RDS → Proxies → Create Proxy**.

**Proxy Configuration:**

| Setting                | Value                   |
| ---------------------- | ----------------------- |
| Engine Family          | MariaDB / MySQL         |
| Proxy Name             | `codecloud-app-dbproxy` |
| Idle Client Timeout    | 30 minutes              |
| Target Group DB        | `codecloud-app-db`      |
| Max Connections        | 100                     |
| TLS                    | Disabled                |
| Secrets Manager Secret | `kk-rds-secret`         |
| IAM Role               | Default                 |
| Proxy Security Group   | `proxy`                 |

Clicked **Create Proxy**.
Wait time: up to **20 minutes**.

Proxy status eventually changed to:

```
Available
```

---

### **Step 8 — Test Connection Through RDS Proxy**

From the EC2 instance:

```bash
mysql -h <proxy-endpoint> -P 3306 -u admin -p
```

Entered password.
Received an error:

```
ERROR 2013: Lost connection to MySQL server during handshake
```

This suggested a **security group misconfiguration**.

On review:

* Proxy SG allowed inbound from EC2 SG
* EC2 SG allowed inbound from RDS SG
* RDS SG allowed inbound from Proxy SG

Chaining looked correct — issue remained unresolved but likely SG-related.

---

## 💡 Notes / Tips

* When using SSH → always `chmod 400` the key pair to avoid permission errors.
* RDS Proxy requires **correct SG chaining**: EC2 → Proxy → RDS.
* If you get MySQL handshake errors, 90% of the time it’s a security group rule mismatch.
* Secrets Manager is preferred over storing passwords on EC2.
* Proxies reduce the overhead of opening/closing DB connections in applications.

---

## 📌 Lab Summary

| Step                          | Status | Key Takeaways                          |
| ----------------------------- | ------ | -------------------------------------- |
| Create MariaDB RDS instance   | ✅      | Configured DB, storage, network        |
| Connect via EC2 SSH           | ✅      | Verified MySQL login                   |
| Create Secrets Manager secret | ✅      | Stored DB credentials securely         |
| Create proxy security group   | ✅      | Correct inbound/outbound SG chaining   |
| Create RDS Proxy              | ✅      | Successfully configured                |
| Test proxy connection         | ⚠️     | Connection failed → SG issue suspected |

---

## 🔗 References

* [Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html)
* [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
* [RDS Proxy Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)
* [MySQL Client Reference](https://dev.mysql.com/doc/refman/8.0/en/mysql.html)
