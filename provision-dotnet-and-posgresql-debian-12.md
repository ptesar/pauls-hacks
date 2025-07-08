# Provision .NET and PostgreSQL on Debian 12 Systems

Step by step guide to provision .NET runtime and PosgreSQL for self sufficient web application hosting on Debian 12 systems.

## Who is it for

Anyone who frequently sets up new application hosts and has memory span of a gold fish like me.

## Disclaimer

While none of the outlined steps modify the underlying file systems, they add new repositories and install new packages. Goes without saying that you should backup everything if the host you are working on has data you can't afford to lose. 

## Dotnet

### 1. Add Microsoft Package Repository 

Run the following commands to add the Microsoft package signing key to your list of trusted keys and add the package repository:

```
wget https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
```

### 2. Update Package Lists

Update local package index by:

```
sudo apt-get update
```

### 3. Install .NET Runtime

Install the package, change the version for your usecase:

```
sudo apt-get install -y aspnetcore-runtime-8.0
```

## PostgreSQL

### 1. Update Package Lists

Update local package index by:

```
sudo apt-get update
```

### 2. Install PostgreSQL

To install PostgreSQL, run the following command in the command prompt:

```
sudo apt install postgresql
```

The database service is automatically configured with viable defaults.


### 3. Configure PostgreSQL

Open the configuration file for editing:

```
sudo nano /etc/postgresql/*/main/postgresql.conf
```

Locate the line `#listen_addresses = 'localhost'`, uncomment the line and change its value to `*` as follows:

```
listen_addresses = '*'
```

Setting `'*'` will configure PostgreSQL to listen on all available network interfaces, both IPv4 and IPv6. To listen on all IPv4 interfaces, set listen_addresses to `'0.0.0.0'`, while `'::'` will listen on all IPv6 interfaces. You can also specify comma-separated list of fixed addresses that the PostgreSQL will bind to.

Restart the service:

```
sudo systemctl restart postgresql
```
### 4. Set DB User Password

Run the following command to connect to the local PostgreSQL instance as default `postgres` user:

```
sudo -u postgres psql
```

With the sql query cli open enter the following query to change the `postgres` user password:

```
ALTER USER postgres with encrypted password 'your_password';
```

Change the value of the password to your preferred value.

Then quit the sql query cli by typing:

```
exit
```

### 4. Allow User Network Access

Edit the client authentication configuration file:

```
sudo nano /etc/postgresql/*/main/pg_hba.conf
```

Add the following line to IPV4 location connections:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
hostssl all             all             0.0.0.0/0               scram-sha-256
```

Add the following line to IPV6 location connections:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
hostssl all             all             ::                      scram-sha-256
```

The resulting configuration should be:

```conf
# Database administrative login by Unix domain socket
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer

# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
hostssl all             all             0.0.0.0/0               scram-sha-256

# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
hostssl all             all             ::/0                    scram-sha-256

# Allow replication connections:
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
```

### 4. Restart PostgreSQL Service

Run the following command:

```
sudo systemctl restart postgresql
```

## Useful links

 - [Install the .NET SDK or the .NET Runtime on Debian](https://learn.microsoft.com/en-us/dotnet/core/install/linux-debian) Official guide from Microsoft with instructions how to install .NET on Debian.
