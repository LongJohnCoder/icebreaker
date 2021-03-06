icebreaker - BETA (stable release in mid March)
------
Break the ice with that cute Active Directory environment over there. Automates network attacks against Active Directory to deliver you plaintext credentials when you're inside the network but outside of the Active Directory environment. Performs 5 different network attacks for plaintext credentials as well as hashes. Autocracks hashes found with JohnTheRipper and a custom 1 million password wordlist specifically for Active Directory passwords.

* RID cycling 
  * Uses Nmap to find NULL SMB sessions
  * Performs asynchronous RID cycling to find valid usernames
  * Performs a 2 password reverse bruteforce of found usernames
  * Passwords tested: P@ssw0rd and \<season\>\<year\>, e.g., Winter2018
* SCF file upload
  * Uses Nmap to find anonymously writeable shares on the network
  * Writes an SCF file to the share with a file icon that points to your machine
  * When a user opens the share in Explorer their hash is sent to you
  * Autocracks the hash with john and top 10 million password list
* LLMNR/NBTNS/mDNS poisoning
  * Uses Responder.py to poison the layer 2 network and capture user hashes
  * Autocracks the hash with john and top 10 million password list
* SMB relay
  * Uses ntlmrelay.py and Responder.py to relay SMB hashes
  * After a successful relay it will do the following on the victim machine:
    * Add an administrative user - icebreaker:P@ssword123456
    * Run an obfuscated and AMSI bypassing version of Mimikatz and parse the output for hashes and passwords
* IPv6 DNS poison
  * Uses mitm6 and ntlmrelayx.py to poison IPv6 DNS and capture user and machine hashes
  * Creates fake WPAD server with authentication
  * Note: this can easily cause network connectivity issues for users so use sparingly


#### How It Works
It will perform the above 5 network attacks in order. RID cycling and SCF file uploads usually go fast, then it lingers on attack 3, Responder.py, for 10 min by default. After that amount of time, or the user-specified amount of time has passed, it will move on to the final two attacks which are run in parallel and indefinitely. 

After performing RID cycling and an asynchronous bruteforce it moves on to upload SCF files to anonymously writeable shares. If an SCF file was successfully uploaded and a user visits that file share in Explorer the user's hash will be captured and attempted to be cracked by icebreaker. If the hash is captured while attack 4, SMB relay, is running, the hash will be relayed for potential command execution. Relaying a hash to another machine allows us to impersonate the user whose hash we captured and if that user has administrative rights to the machine we relayed the hash to then we can perform command execution.

Once ntlmrelayx relays a captured hash it will run a base64-encoded powershell command that first adds an administrative user (icebreaker:P@ssword123456) then runs an obfuscated and AMSI-bypassing version of Mimikatz. This mimikatz output is parsed and delivered to the user in the standard output as well as in the found-passwords.txt file if any plaintext passwords or NTLM hashes are found. 

If icebreaker is run with the --auto flag, then upon reaching attack 4 icebreaker will run [Empire](https://www.powershellempire.com/) and [DeathStar](https://byt3bl33d3r.github.io/automating-the-empire-with-the-death-star-getting-domain-admin-with-a-push-of-a-button.html) in xterm windows. With this option, instead of running mimikatz on the remote box that we relayed the hash to, icebreaker add an administrative user and right after that it'll run Empire's powershell launcher code to get an agent on the remote machine. DeathStar will use this agent to automate the process of acheiving domain admin. The Empire and DeathStar xterm windows will not close when you exit icebreaker.

Password cracking is done with JohnTheRipper and a custom wordlist. The origin of this list is from the [merged.txt](https://github.com/danielmiessler/SecLists/blob/601038eb4ea18c97177b43a757286d3c8a815db8/Passwords/merged.txt.tar.gz) which is every password from the SecLists GitHub account combined. The wordlist was pruned and includes no passwords with: all lowercase, all uppercase, all symbols, less than 7 characters, more than 32 characters. These rules conform to the default Active Directory password requirements and brought the list from 20 million to just over 1 million which makes password cracking extremely fast.

Note about attack 5, IPv6 DNS poisoning: this attack is prone to causing issues on the network. It often causes certificate errors on client machines in the browser. It'll also likely slow the network down. The beauty of this attack is that Windows AD environments are vulnerable by default (are there more beautiful words in the english language than "vulnerable by default"?) but you should be wary of side-effects.

#### Installation
As root:
```
./setup.sh
pipenv shell
```

#### Usage
Run as root.
Read from a newline separated list of IP addresses (single IPs or CIDR ranges) and instead of having ntlmrelayx add a user and mimikatz the victim upon hash relay, have it execute a custom command on the victim machine. 

```./icebreaker -l targets.txt -c "net user /add User1 P@ssw0rd"```

Read from Nmap XML file and tell Responder to use the eth0 interface rather than the default gateway interface

```./icebreaker -x nmapscan.xml -i eth0```

Skip all five attacks and don't autocrack hashes

```./icebreaker.py -x nmapscan.xml -s rid,scf,llmnr,ntlmrelay,dns,crack```

Run attack 3, LLMNR poisoning, for 30 minutes before moving on to attack 4, SMB relaying and use a custom password list for attack 1's reverse bruteforce

```./icebreaker.py -x nmapscan.xml -t 30 -p /home/user/password-list.txt```

My favorite usage - input targets file, skip Responder's LLMNR/NBT-NS/mDNS poisoning, and run Empire and DeathStar once attack 4 starts

```./icebreaker.py -l targets.txt -s llmnr --auto```

