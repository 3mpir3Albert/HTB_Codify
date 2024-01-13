# HTB_Codify
In this repository you can find the Codify server write up and the exploit used to make the privilege escalation.
## Write up
First of all, before starting to explain how has been pwned Codify, I'm going to show you all the concepts that are used to vulnerate this server:
  - An NodeJs vulnerability is exploited to gain access as user CVS (user with minimum privileges)
  - An exaustive system enumeration is needed to detect an .db file.
  - Cracking hashes.
  - Abusing SUDOERS privilege.
