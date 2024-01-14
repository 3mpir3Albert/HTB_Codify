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

<div style="text-align:center">
  [![hosts.png](https://i.postimg.cc/KvyMrT1X/hosts.png)](https://postimg.cc/7bmhqf1X)
</div>
