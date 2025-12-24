### 1. LinuxCLI - Shells Basics

# Solution:

- This part was just making use of simple linux commands to reach desired objectives plus some trivia
- It included shell scripts and using `ls -a` to find hidden files and piping.

***
### 2. Phishing - Merry clickmas

# Solution:
- This was a tutorial on the social engineering toolkit, we used SET to send mass phishing emails to the recipient `factory@wareville.thm`.
- We had already set up a realistic looking server that would catch the username and the password, to complete the challenge we had to draft a convincing phishing email that would guide the recipient to our website.
- We obtained the password from our phishing attack and then used that password to gain access to comapnies critical information on the ammout of toys they are making, this was lowkenuinely fun.

***
### 3. Splunk Basics - Did you SIEM?

# Soltion:
- We used splunk to analyse th logs of an infected machine to try and find out the attack.
- The first part of the challenge was familiarisation with the UI and basic queries like `index=main`.
- We then inserted a query to find out the the time of hte attack we see a surge in activity from `10-10-2025` to `14-10-2025` with the activity peaking at `12-10-2025`.
- We started searching for anomalies and found three interesting results a user named `Havij` and ip `198.51.100.55`
  and in path being requested.
- We narrowed down the attacker's ip with this query `sourcetype=web_traffic user_agent!=*Mozilla* user_agent!=*Chrome* user_agent!=*Safari* user_agent!=*Firefox* | stats count by client_ip | sort -count | head 5`
- After narrowing down on the attack chain we found out the attacker used low-level commands like `curl/wgets`.
- The attacker is trying to request `/etc/psswd` and trying to redirect us to some `evil.site`.
- Using this query we succesfully found out the use of `SQL injection` 
```
sourcetype=web_traffic client_ip="<REDACTED>" AND user_agent IN ("*sqlmap*", "*Havij*") | table _time, path, status
```
A status code of 504 confirms a succesful time baseed sql injection attack.
- The execution of `/shell.php?cmd=./bunnylock.bin` indicates a ransomware like program executed on the server
- We then used this query to confirm that the malware established a connection to attackers C2
```
sourcetype=firewall_logs src_ip="10.10.1.5" AND dest_ip="<REDACTED>" AND action="ALLOWED" | table _time, action, protocol, src_ip, dest_ip, dest_port, reason
```
