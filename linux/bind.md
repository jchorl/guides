# BIND
[bind](https://www.isc.org/bind/) is quite the beast.

## Setting up BIND with static config
1. Provision an instance on AWS/GCP
1. `apt-get update && apt-get install -y bind9`
1. Create a log dir

    ```bash
    j@bind:~$ sudo mkdir -p /var/log/named
    j@bind:~$ sudo chown bind:bind /var/log/named
    ```
    _NOTE: `named` sets up an AppArmor config in `/etc/apparmor.d/usr.sbin.named`. Writing logs to any other dir will require updating the AppArmor config._
1. Add zone and logging to `/etc/bind/named.conf.local`

    ```bash
    j@bind:~$ cat /etc/bind/named.conf.local
    // ...

    zone "choo.dev" {
      type master;
      file "/etc/bind/zones/db.choo.dev";
    };

    logging {
      channel query.log {
        file "/var/log/named/query.log";
        severity info;
      };

      category queries { query.log; };
    };
    ```
1. Write a config

    ```bash
    j@bind:~$ sudo cp /etc/bind/db.local /etc/bind/zones/db.choo.dev
    j@bind:~$ cat /etc/bind/zones/db.choo.dev
    $TTL    604800
    @       IN      SOA     ns1.choo.dev. admin.choo.dev. (
                                  3         ; Serial
                             604800         ; Refresh
                              86400         ; Retry
                            2419200         ; Expire
                             604800 )       ; Negative Cache TTL
    ;
    ; name servers - NS records
            IN      NS      ns1.choo.dev.
    ; name servers - A records
    ns1.choo.dev.   IN      A       34.82.193.142
    ; choo.dev - A records
    hi.choo.dev.    IN      A       10.128.100.101
    ```
1. Restart BIND

    ```bash
    j@bind:~$ sudo systemctl restart bind9
    ```
1. Check it

    ```bash
    j@bind:~$ dig @127.0.0.1 hi.choo.dev
    ...

    ;; ANSWER SECTION:
    hi.choo.dev.            604800  IN      A       10.128.100.101
    ```

## Setting up BIND with dynamic config
1. Creating the db

   Foremost, BIND will be dynamically updating config. According to its AppArmor config, it can write to `/var/lib/bind`. So we'll store config there.

   Starting with `/etc/bind/db.empty`, we can make some modifications:
   ```bash
   j@bind:~$ sudo cp /etc/bind/db.empty /var/lib/bind/db.choo.dev
   j@bind:~$ cat /var/lib/bind/db.choo.dev
   $TTL    86400
   @       IN      SOA     choo.dev. admin.choo.dev. (
                                 1         ; Serial
                            604800         ; Refresh
                             86400         ; Retry
                           2419200         ; Expire
                             86400 )       ; Negative Cache TTL
   ;
   @       IN      NS      ns1.choo.dev.
   ns1     IN      A       34.82.193.142
   ```

1. Allow modifications to the db
    ```bash
    j@bind:~$ cat /etc/bind/named.conf.local
    ...
    zone "choo.dev" {
      type master;
      file "/var/lib/bind/db.choo.dev";
      allow-update { 127.0.0.1; };
    };
    ...
    ```

1. Add logging
    ```bash
    j@bind:~$ cat /etc/bind/named.conf.local
    ...
    logging {
      ...
      channel update.log {
        file "/var/log/named/update.log";
        severity info;
      };

      category update { update.log; };
    };
    ...
    ```

### Testing
There is a protocol for updating dns records. `nsupdate` is a handy util to speak that protocol. BIND speaks this protocol.

```bash
j@bind:~$ sudo nsupdate -l -4
> update add *.choo.dev 86400 A 172.16.1.1
> send

j@bind:~$ dig @127.0.0.1 hi.choo.dev
...
;; ANSWER SECTION:
hi.choo.dev.            86400   IN      A       172.16.1.1
...
```

## Tips
### Wildcards
You can put wildcards in configs, i.e.
```
*.choo.dev.     IN      A       10.128.100.101
```

### Validate configs
```bash
j@bind:~$ sudo named-checkconf
j@bind:~$ echo $?
0
j@bind:~$ sudo named-checkzone choo.dev /etc/bind/zones/db.choo.dev
zone choo.dev/IN: loaded serial 3
OK
```

### Command files
You can also use files for `nsupdate`
```bash
j@bind:~$ cat update.txt 
update delete *.choo.dev A
update add *.choo.dev 86400 A 172.16.8.1
send
j@bind:~$ sudo nsupdate -l -4 update.txt
```
