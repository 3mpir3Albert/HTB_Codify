# HTB_Codify
In this repository you can find the Codify server write up and the exploit used to make the privilege escalation.
## Write up
First of all, before starting to explain how has been pwned Codify, I'm going to show you all the concepts that are used to vulnerate this server:
  - An NodeJs vulnerability is exploited to gain access as user CVS (user with minimum privileges)
  - An exaustive system enumeration is needed to detect an ''.db'' file.
  - Cracking hashes.
  - Abusing SUDOERS privilege.

As usual, reconnaissance phase was the first phase executed due to the fact that a hacker needs to know as much information about his target as possible before starting to exploit posible vulnerabilities.

To do this, it is necessary to add a new line in the file ''/etc/hosts'', where the server IP is defined when a request is made to codify.htb:

[![hosts.png](https://i.postimg.cc/KvyMrT1X/hosts.png)](https://postimg.cc/7bmhqf1X)

Next, Nmap is used to scan all ports on the server. The tool output is as follows:

[![Nmap.png](https://i.postimg.cc/4ySGNV78/Nmap.png)](https://postimg.cc/XBdmsZgF)

As seen in the last image, three open ports were discovered, two HTTP ports and one SSH port. The SSH port had a fairly up-to-date version, so it would be difficult to exploit this service. On the other hand, there are two http ports, so it is worth scanning the technology that is working behind both ports:

[![Scann.png](https://i.postimg.cc/qqLN1mdv/Scann.png)](https://postimg.cc/4HY4dQdD)

Practically, the output of both ports is the same, so it is likely that the same website is being exposed on both ports. For this reason, the two websites are accessed and verified to be the same website.

Searching for information about the website, at "http://codify.htb/about" one discovers that it makes use of the NodeJs VM2 library. More specifically, it uses VM2 version 3.9.16:

[![VM2.png](https://i.postimg.cc/cHNCKZ0P/VM2.png)](https://postimg.cc/SXDk5BXG)

VM2 is a JavaScript library used to virtualize a sandbox on a server. In this case, this sandbox is used to run insecure code given by a user in a safe way. For this purpose, VM2 disables some global objects such as process, which allows to obtain information and control the main process.

In addition, developers disabled other modules such as 'child_process' and 'fs'. The child_process module allows to generate threads and thus execute commands on the server, and the fs module allows to interact directly with the file system.
