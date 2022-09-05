# Debian OpenSSL Predictable PRNG Toys 


![DebianCartoon](SalvagedArchives/DebianCartoon.png)

### The Bug
On May 13th, 2008 the Debian project [announced](http://www.debian.org/security/2008/dsa-1571) that Luciano Bello found an interesting vulnerability in the OpenSSL package they were distributing. The bug in question was caused by the removal of the following line of code from md_rand.c
```c
  MD_Update(&m;,buf,j);
  [ .. ]
  MD_Update(&m;,buf,j); /* purify complains */
```
These lines were [removed](http://svn.debian.org/viewsvn/pkg-openssl/openssl/trunk/rand/md_rand.c?rev=141&view=diff&r1=141&r2=140&p1=openssl/trunk/rand/md_rand.c&p2=/openssl/trunk/rand/md_rand.c) because they caused the [Valgrind](http://valgrind.org/) and Purify tools to produce warnings about the use of uninitialized data in any code that was linked to OpenSSL. You can see one such report to the OpenSSL team [here](SalvagedArchives/Avoid-uninitialized-data-in-random-buffer.jpg). Removing this code has the side effect of crippling the seeding process for the OpenSSL PRNG. Instead of mixing in random data for the initial seed, the only "random" value that was used was the current process ID. On the Linux platform, the default maximum process ID is 32,768, resulting in a very small number of seed values being used for all PRNG operations.
### The Impact
All SSL and SSH keys generated on Debian-based systems (Ubuntu, Kubuntu, etc) between September 2006 and May 13th, 2008 may be affected. In the case of SSL keys, all generated certificates will be need to recreated and sent off to the Certificate Authority to sign. Any Certificate Authority keys generated on a Debian-based system will need be regenerated and revoked. All system administrators that allow users to access their servers with SSH and public key authentication need to audit those keys to see if any of them were created on a vulnerabile system. Any tools that relied on OpenSSL's PRNG to secure the data they transferred may be vulnerable to an offline attack. Any SSH server that uses a host key generated by a flawed system is subject to traffic decryption and a man-in-the-middle attack would be invisible to the users. This flaw is ugly because even systems that do not use the Debian software need to be audited in case any key is being used that was created on a Debian system. The Debian and Ubuntu projects have released a set of tools for identifying vulnerable keys. You can find these listed in the references section below.
#### References
- [Debian Wiki (Best Resource)](http://web.archive.org/web/20110718002842/http://wiki.debian.org/SSLkeys)
- [CVE-2008-0166](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-0166)
- [BID-29179](http://web.archive.org/web/20111012065321/http://www.securityfocus.com/bid/29179)
- [Debian OpenSSL Patch dsa-1571](https://www.debian.org/security/2008/dsa-1571)
- [Debian OpenSSH Patch dsa-1576](https://www.debian.org/security/2008/dsa-1576)
#### Vendor Tools
- [OpenSSL Key Blacklist](https://launchpad.net/ubuntu/+source/openssl-blacklist)
- [OpenSSH Key Blacklist](https://launchpad.net/ubuntu/+source/openssh-blacklist)
- [OpenVPN Key Blacklist](https://launchpad.net/ubuntu/+source/openvpn-blacklist)
#### Security Tools
- [Hubert Seiwert's Remote Host Key Scanner](SalvagedArchives/Remote-host-key-scanner-for-Debian-SSH-updated-2008-05-16.md)
- Sebastian Spitzner's Remote Host Key Scanner (v1)
- [Markus Mueller's Brute Force Perl Script](SalvagedArchives/Markus-Mueller's-Brute-Force-Perl-Script.txt)
- L4teral's Brute Force Ruby Script
- [CR0.org's Debian SSH/SSL Tools](https://www.cr0.org/progs/sshfun/)
- Bootable SSL Certificate Generator (ISO)

### The Toys
The blacklists published by Debian and Ubuntu demonstrate just how small the key space is. When creating a new OpenSSH key, there are only 32,767 possible outcomes for a given architecture, key size, and key type. The reason is that the only "random" data being used by the PRNG is the ID of the process. In order to generate the actual keys that match these blacklists, we need a system containing the correct binaries for the target platform and a way to generate keys with a specific process ID. To solve the process ID issue, I wrote a [shared library](http://metasploit.com/users/hdm/tools/getpid-preload.tar.gz) that could be preloaded and that returns a user-specified value for the getpid() libc call.

The next step was to build a chroot environment that contained the actual binaries and libraries from a vulnerable system. I took a snapshot from a Ubuntu system on the local network. You can find the entire chroot environment [here](http://metasploit.com/users/hdm/tools/debian-openssl/ubunturoot.tar.bz2) In order to generate an OpenSSH key with a specific type, bit count, and process ID, I wrote a shell script that could be executed from within the chroot environment. You can find this shell script [here](http://metasploit.com/users/hdm/tools/debian-openssl/dokeygen.sh). This script is placed into the root directory of the extracted Ubuntu filesystem. In order to generate a key, this script is called with the following command line:
```shell
 # chroot ubunturoot /dokeygen.sh 1 -t dsa -b 1024 -f /tmp/dsa_1024_1
```
This will generate a new OpenSSH 1024-bit DSA key with the value of getpid() always returning the number "1". We now have our first pre-generated SSH key. If we continue this process for all PIDs up to 32,767 and then repeat it for 2048-bit RSA keys, we have covered the valid key ranges for x86 systems running the buggy version of the OpenSSL library. With this key set, we can compromise any user account that has a vulnerable key listed in the authorized_keys file. This key set is also useful for decrypting a previously-captured SSH session, if the SSH server was using a vulnerable host key. Links to the pregenerated key sets for 1024-bit DSA and 2048-bit RSA keys (x86) are provided in the downloads section below.

The interesting thing about these keys is how they are tied to the process ID. Since most Debian-based systems use sequential process ID values (incrementing from system boot and wrapping back around as needed), the process ID of a given key can also indicate how soon from the system boot that key was generated. If we look at the inverse of that, we can determine which keys to use during a brute force based on the target we are attacking. When attempting to guess a key generated at boot time (like a SSH host key), those keys with PID values less than 200 would be the best choices for a brute force. When attacking a user-generated key, we can assume that most of the valid user keys were created with a process ID greater than 500 and less than 10,000. This optimization can significantly speed up a brute force attack on a remote user account over the SSH protocol.

In the near future, this site will be updated to include a brute force tool that can be used quickly gain access to any SSH account that allows public key authentication using a vulnerable key. The keys in the data files below use the following naming convention:
```shell
 / Algorithm / Bits / Fingerprint-ProcessID
   and
 / Algorithm / Bits / Fingerprint-ProcessID.pub  
```
To obtain the private key file for any given public key, you need to know the key fingerprint. The easiest way to obtain this fingerprint is through the following command:
```shell
 $ ssh-keygen -l -f targetkey.pub
 2048 c6:7b:14:fa:ae:b6:89:e6:67:17:ee:04:17:b0:ec:4e targetkey.pub
```
If we look at the public key in an editor, we can also infer that the key type is RSA. In order to locate the private key for this public key, we need to extract the data files, and look for a file named:
```shell
 rsa/2048/c67b14faaeb689e66717ee0417b0ec4e-26670
```
In the example above, the fingerprint is represented in hexadecimal with the colons removed, and the process ID is indicated as "26670". If we want to authenticate to a vulnerable system that uses this public key for authentication, we would run the following command:
```shell
 $ ssh -i rsa/2048/c67b14faaeb689e66717ee0417b0ec4e-26670 root@targetmachine
```
#### Our Tools
- GetPID Faker Shared Library (4.0K)
- Ubuntu Root Filesystem (4.9M)
- Key Generation Script (8.0K)
#### Common Keys
- SSH 1024-bit DSA Keys X86 (30.0M)
- SSH 2048-bit RSA Keys X86 (48.0M)
#### Uncommon Keys
- SSH 1023-bit RSA Keys X86 (25.0M)
- SSH 1024-bit RSA Keys X86 (26.0M)
- SSH 2047-bit RSA Keys X86 (48.0M)
- SSH 4096-bit RSA Keys X86 (94.0M)
- SSH 8192-bit RSA Keys X86 (PID 1 to 4100+) (29.0M)
- SSH RSA1 Keys X86 (139.0M)

### Frequently Asked Questions
Q: How long did it take to generate these keys?
A: I used 31 Xeon cores clocked at 2.33Ghz. It took two hours to generate the 1024-bit DSA and 2048-bit RSA keys for x86. The 4096-bit RSA keys took about 6 hours to generate. The 8192-bit RSA key generation would take about 100 hours at its current rate and will likely be stopped before completion.

Q: Will you share your code for distributing the key-generation across mulitple processors?
A: Nope. The code is hardcoded for this specific cluster and is too poorly-written to be worth cleaning up.

Q: How long does it take a crack a SSH user account using these keys?
A: This depends on the speed of the network and the configuration of the SSH server. It should be possible to try all 32,767 keys of both DSA-1024 and RSA-2048 within a couple hours, but be careful of anti-brute-force scripts on the target server.

Q: I use 16384-bit RSA keys, can these be broken?
A: Yes, its just a matter of time and processing power. The 8192-bit RSA keyset would take about 3100 hours of CPU time to generate all 32,767 keys (100 hours on the 31 cores im using now). I imagine the 16384-bit RSA keyset would take closer to 100,000 hours of CPU time. One thing to keep in mind is that most keys are within a much smaller range, based on the process ID seed, and the entire set would not need to be generated to cover the majority of user keys (most keys are within the first 3,000 process IDs).

Copyright © 2008 H D Moore 
