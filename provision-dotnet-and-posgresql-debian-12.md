# Provision .NET and PostgreSQL on Debian 12 Systems

Step by step guide to provision .NET runtime and PosgreSQL for self sufficient web application hosting on Debian 12 systems.

## Who is it for

Anyone who frequently sets up new application hosts and has memory span of a gold fish like me.

## Disclaimer

While none of the outlined steps modify the underlying file systems, they add new repositories and install new packages. Goes without saying that you should backup everything if the host you are working on has data you can't afford to lose. 

## Dotnet

To host .NET apps you need to download and install .NET runtime.

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

If you intend to host PostgreSQL databases on the same server as your .NET apps you need to install and configure PostgreSQL server. This section is optional if you have an existing PostgreSQL server on the network or you plan to provision a new host for the databases. 

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

Be aware that configuring the listen address to bind to all available networks will make the PostgreSQL server accessible from anywhere. If this is not your intention, **access should be restricted using a firewall** either directly on the host operating system or at the nearest network router.

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

### 5. Allow User Network Access

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
hostssl all             all             ::/0                    scram-sha-256
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

Note that allowing the user network access to `0.0.0.0/0` and `::/0` with `all` directives will permit all database users to connect to the databases they have granted access through roles set within PosgreSql server instance.

### 6. Restart PostgreSQL Service

Run the following command:

```
sudo systemctl restart postgresql
```

## Set Up .NET System User

It's a recommended practice to limit the permission scope of running hosted apps with a dedicated os user account. I tend to stick to convention of naming the user `dotnet-data` for clarity. Crate the new user by typing:

```
sudo useradd dotnet-data
```

## Set Up .NET App Folder

While you can store your dotnet apps anywhere, I tend to stick to convention of storing all dotnet apps in the `/var/dotnet/` folder. 

### 1. Create Folder

Create the folder by typing:

```
sudo mkdir /var/dotnet/
```

### 2. Change Folder Owner

Change folder ownership by typing:

```
sudo chgrp -R dotnet-data /var/dotnet/
sudo chown -R dotnet-data /var/dotnet/
```

### 3. Set Folder Permissions

Set folder permissions by typing:

```
sudo chmod -R 770 /var/dotnet/
```

## Automate .NET User File Ownership

To make sure any file or folder we create in our .NET app folder `/var/dotnet/` gets automatically owned by our .NET user `dotnet-data` we can use `inotify` to monitor all nested folders and files for changes and set the ownership of the files for us automatically. Having the files automatically owned by the same system user addresses the permission issues and annoyances with deploys and file uploads.

### 1. Install Incron Service

Install it with command:

```
sudo apt-get install incron
```

### 2. Allow Root User

Allow root to use incron by editing `/etc/incron.allow` with:

```
sudo nano /etc/incron.allow
```

and add `root` to the file, then exit by pressing `[ctrl]+[x]` and save by pressing `[s]`.

### 3. Define Folder Monitoring And Action 

Edit your incrontab with:

```
sudo incrontab -u root -e
```

Add the following line to it:

```
/var/www/html IN_CREATE /bin/chown -R dotnet-data:dotnet-data /var/www/dotnet/
```

Save and exit.

Now as soon as a file is created in the `/var/dotnet/` directory incron will automatically set ownership to `www-dotnet` user and group.


## Useful links

 - [Install the .NET SDK or the .NET Runtime on Debian](https://learn.microsoft.com/en-us/dotnet/core/install/linux-debian) Official guide from Microsoft with instructions how to install .NET on Debian.
 - [Ask Ubuntu: Make owner of newly create files AND folders](https://askubuntu.com/a/631316) Comprehensive response by [krt](https://askubuntu.com/users/374354/krt) with instructions on how to set up incron.
