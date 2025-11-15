# MySQL Client-Server Setup on AWS EC2

This guide explains how to set up MySQL in a client-server architecture between two EC2 instances in the same AWS VPC.

<img width="1402" height="346" alt="Screenshot from 2025-11-15 10-55-34" src="https://github.com/user-attachments/assets/d87b966e-0046-4c96-85e1-1a05b6eac28d" />

## Architecture Overview

- **MySQL Server EC2**: `172.31.36.239` - Hosts the MySQL database
- **MySQL Client EC2**: `172.31.43.109` - Connects to the server remotely
- **Network**: Both instances are in the same VPC/subnet
<img width="1848" height="972" alt="Screenshot from 2025-11-15 09-36-58" src="https://github.com/user-attachments/assets/bdbc20b6-5dca-4e30-a330-7eecc68ba915" />


## Step-by-Step Setup

### 1. MySQL Server Configuration

#### Install MySQL Server
```bash
sudo apt update
sudo apt install mysql-server
```

#### Configure MySQL for Remote Access
Edit MySQL configuration:
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Change bind address:
```ini
bind-address = 0.0.0.0
```

#### Create Remote User
```bash
sudo mysql -u root -p
```

```sql
-- Create user for client connection
CREATE USER 'remote_user'@'172.31.43.109' 
IDENTIFIED WITH mysql_native_password BY 'SecurePass123!';

-- Grant privileges
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'172.31.43.109';

FLUSH PRIVILEGES;
EXIT;
```

#### Configure AWS Security Group
Add inbound rule to MySQL server's security group:
- Type: MySQL/Aurora
- Protocol: TCP
- Port: 3306
- Source: Client's private IP (`172.31.43.109/32`) or security group

#### Restart MySQL Service
```bash
sudo systemctl restart mysql
sudo systemctl status mysql
```

### 2. MySQL Client Configuration

#### Install MySQL Client
```bash
sudo apt update
sudo apt install mysql-client
```

#### Test Connection
```bash
mysql -h 172.31.36.239 -u remote_user -p
```

### 3. Troubleshooting Common Issues

#### Authentication Plugin Issues
MySQL 8.0+ uses `caching_sha2_password` which can cause connection problems. Use `mysql_native_password` instead:

```sql
ALTER USER 'remote_user'@'172.31.43.109' 
IDENTIFIED WITH mysql_native_password BY 'SecurePass123!';
```

#### Network Connectivity Test
```bash
# Test if MySQL port is accessible
telnet 172.31.36.239 3306

# Check basic connectivity
ping 172.31.36.239
```

#### User Management
View existing users:
```sql
SELECT user, host, plugin FROM mysql.user WHERE user = 'remote_user';
```

## Important Notes

### User Host Specifications
- `remote_user@172.31.43.109` - Specific IP access
- `remote_user@localhost` - Local server access only
- `remote_user@'172.31.%'` - Subnet-wide access

### Password Requirements
- Use strong passwords (12+ characters)
- Mix uppercase, lowercase, numbers, and special characters
- Avoid common words or simple patterns

### Security Considerations
1. Use private IP addresses for internal communication
2. Restrict security group rules to specific IPs
3. Use `mysql_native_password` for better compatibility
4. Regularly update passwords and privileges

## Verification Steps

### On Server
```sql
-- Verify user exists with correct privileges
SELECT user, host, authentication_string FROM mysql.user;
SHOW GRANTS FOR 'remote_user'@'172.31.43.109';
```

### On Client
```bash
# Test connection
mysql -h 172.31.36.239 -u remote_user -p -e "SHOW DATABASES;"
```

## Useful MySQL Commands

### Connection Options
```bash
# Basic connection
mysql -h hostname -u username -p

# Connect to specific database
mysql -h hostname -u username -p database_name

# Execute single command
mysql -h hostname -u username -p -e "SHOW DATABASES;"
```

### Common MySQL Commands
```sql
-- Show databases
SHOW DATABASES;

-- Create database
CREATE DATABASE test_db;

-- Use database
USE database_name;

-- Show tables
SHOW TABLES;

-- Exit MySQL
EXIT;
```

## Troubleshooting Matrix

| Issue | Symptom | Solution |
|-------|---------|----------|
| Connection refused | "Can't connect to MySQL server" | Check security groups, bind-address, MySQL service status |
| Access denied | "Access denied for user" | Verify user privileges, password, host specification |
| Authentication plugin | "caching_sha2_password" timeout | Use mysql_native_password plugin |
| Network issues | Telnet timeout | Check VPC routing, NACLs, security groups |

This setup enables secure MySQL communication between EC2 instances using private IP addresses within the same VPC.
