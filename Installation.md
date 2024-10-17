## Voiceware Installation

### FreeSWITCH installation

#### Install pre-requisites 

```
apt install -y postgresql-13 python3-pip nfs-common python3-pip cmake sngrep build-essential postgresql-server-dev-13 fail2ban  gnupg2 wget lsb-release
pip3 install psycopg
pip3 install psycopg2
pip3 install redis
pip3 install mongoengine
pip3 install openai
pip3 install deepgram-sdk
pip3 install elevenlabs

```

#### Pull voiceware code
cd /root
git init
git config --global credential.helper store
git clone https://github.com/Nexis-Comms/voiceware.git


#### Setup postgres-prefix extension
```bash
# Debian11-Bullseye
curl -LJO https://github.com/dimitri/prefix/archive/refs/tags/v1.2.9.tar.gz
tar -xvzf prefix-1.2.9.tar.gz
cd prefix-1.2.9/
make
make install
cd ..
rm -rf prefix-1.2.9*
runuser -u postgres --  psql -c "create extension if not exists \"prefix\";" freeswitch

#### Setup Database

```bash
runuser -u postgres --  psql -c "CREATE ROLE freeswitch WITH LOGIN SUPERUSER PASSWORD 'Fr33sw2tc#';"
runuser -u postgres --  psql -c "CREATE DATABASE freeswitch;"
runuser -u postgres --  psql -f "/root/voiceware/freeswitch/db_queries.sql" freeswitch
# Supposed to have postgresql version 13
echo "listen_addresses='*'" >> /etc/postgresql/13/main/postgresql.conf
echo "host all all 0.0.0.0/0 md5" >> /etc/postgresql/13/main/pg_hba.conf
systemctl restart postgresql

```
#### Set up nfs shares
```bash
mkdir /mnt/nfs_clientshare -p
echo "${SERVER_IP}:/mnt/nfs_share /mnt/nfs_clientshare nfs defaults 0 0" >> /etc/fstab
mount -a
```

#### Install FreeSWITCH

```bash
# Debian11-Bullseye

TOKEN=pat_6Pa56zJaXAkdrixrJnSrjadk

wget --http-user=signalwire --http-password=$TOKEN -O /usr/share/keyrings/signalwire-freeswitch-repo.gpg https://freeswitch.signalwire.com/repo/deb/debian-release/signalwire-freeswitch-repo.gpg

echo "machine freeswitch.signalwire.com login signalwire password $TOKEN" > /etc/apt/auth.conf
chmod 600 /etc/apt/auth.conf
echo "deb [signed-by=/usr/share/keyrings/signalwire-freeswitch-repo.gpg] https://freeswitch.signalwire.com/repo/deb/debian-release/ `lsb_release -sc` main" > /etc/apt/sources.list.d/freeswitch.list
echo "deb-src [signed-by=/usr/share/keyrings/signalwire-freeswitch-repo.gpg] https://freeswitch.signalwire.com/repo/deb/debian-release/ `lsb_release -sc` main" >> /etc/apt/sources.list.d/freeswitch.list

# you may want to populate /etc/freeswitch at this point.
mkdir -p /etc/freeswitch
cp -r /root/voiceware/freeswitch/conf/* /etc/freeswitch/
chown -R freeswitch:freeswitch /etc/freeswitch

# if /etc/freeswitch does not exist, the standard vanilla configuration is deployed
apt-get update && apt-get install -y freeswitch-meta-all

mkdir -p /usr/local/freeswitch/sounds


```

#### Install FS Scripts

```bash
ln -s "$PWD/scripts" /usr/lib/python3/dist-packages/fscripts

mkdir -p /usr/share/freeswitch/scripts/py_scripts/
mkdir -p /usr/lib/python3/dist-packages/py_scripts/
cp -r ./scripts/* /usr/share/freeswitch/scripts/py_scripts/
cp -r ./scripts/* /usr/lib/python3/dist-packages/py_scripts/

```

#### Install mod_bcg729

```bash
apt install libfreeswitch-dev -y
git clone https://github.com/xadhoom/mod_bcg729.git
cd mod_bcg729 || exit
make
make install
cd ..
rm -rf mod_bcg729
```


#### Install fail2ban

```bash
cd /root/voiceware/fail2ban/
touch /var/log/manban.log
cp filter.d/* /etc/fail2ban/filter.d/
cp jail.d/* /etc/fail2ban/jail.d/
systemctl restart fail2ban
systemctl enable fail2ban
cp banip /bin/
cp unbanip /bin/
```

#### Copy certs
```bash
cat /root/voiceware/certs/fullchain.pem > /etc/freeswitch/tls/wss.pem
cat /root/voiceware/certs/cert.pem >> /etc/freeswitch/tls/wss.pem
cat /root/voiceware/certs/privkey.pem >> /etc/freeswitch/tls/wss.pem
```

### update vars.xml

Update the Ip information in the following lines in /etc/freeswitch/vars.xml file.
```xml
<X-PRE-PROCESS cmd="set" data="webapp_pg_ip=1.1.1.1"/>
<X-PRE-PROCESS cmd="set" data="mongo_ip=1.1.1.1"/>
<X-PRE-PROCESS cmd="set" data="sipcapture_ip=1.1.1.1"/>
```
#### Copy ssh keys 
```bash
cat ./webapp_ssh.key >> /root/.ssh/authorized_keys
```
#### 


#### restart fs
```bash
systemctl restart freeswitch

```
