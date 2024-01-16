# Installing Apache NiFi on Centos

## Installing JDK
- Download JDK RPM https://www.oracle.com/java/technologies/downloads/
- Choose RPM Package, here is an example:
```
https://download.oracle.com/java/21/latest/jdk-21_linux-x64_bin.rpm
```
- Install it using below command
```
sudo yum localinstall jdk-21_linux-x64_bin.rpm
```
- Run version command to ensure java is properly installed
```
java -version
```

## Installing Apache NiFi
- Go to Apache NiFi Download page
```
https://nifi.apache.org/download/
```
- Download NiFi standard binaries of the latest stable release
https://dlcdn.apache.org/nifi/1.24.0/nifi-1.24.0-bin.zip
- Use `wget` command to download the software directly on the server
```
wget https://dlcdn.apache.org/nifi/1.24.0/nifi-1.24.0-bin.zip
```
- unzip it
```
unzip nifi-1.24.0-bin.zip
```
- Move extracted package to /opt directory
- Install NiFi as a service
```
/opt/nifi/bin/nifi.sh install nifi
```
- Now we can manage NiFi using `systemctl` commands
### Start NiFi Services
```
systemctl start nifi
```
### Stop NiFi Services
```
systemctl start nifi
```
### Check Status of NiFi Services
```
systemctl status nifi
```
