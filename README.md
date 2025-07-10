# Protocols-and-Servers
Manual protocol exploitation using Telnet and FTP to analyze HTTP, FTP, SMTP, POP3, and IMAP behaviors and uncover plaintext vulnerabilities.

By Ramyar Daneshgar


In this lab I explored how legacy network protocols function at the application layer, using Telnet and FTP clients to connect directly to common services such as HTTP, FTP, SMTP, POP3, and IMAP. These protocols are widely deprecated in secure environments, but many systems - especially in small businesses or legacy environments—still rely on them unknowingly. This exercise allowed me to simulate real-world threat scenarios, such as credential sniffing, and to understand how protocol behavior affects enterprise security posture.

---

## Telnet – Remote Terminal Over Cleartext (Port 23)

I began by connecting to the target using Telnet:

```bash
telnet <target-ip> 23
```

Once connected, I was prompted for a username (`frank`) and password (`D2xc9CgD`). After authentication, I received a shell prompt:

```
frank@bento:~$
```

At this point, I had remote terminal access—without any form of encryption or session hardening.

**Key Takeaway**: Telnet provides direct terminal access over TCP, but all credentials and commands are visible to anyone monitoring the network. This is especially dangerous in shared networks or environments lacking internal segmentation.

---

## HTTP – Manual Page Request via Telnet (Port 80)

To emulate a browser manually, I connected to the HTTP server:

```bash
telnet <target-ip> 80
```

And then entered the raw HTTP request:

```
GET /flag.thm HTTP/1.1
Host: telnet

```

This returned the contents of the target file:

**Flag**: `THM{e3eb0a1df437f3f97a64aca5952c8ea0}`

By constructing this request manually, I was able to retrieve the resource directly without using a browser.

**Context**: This illustrates that HTTP requires minimal structure to be functional. Browsers automate these calls, but when using tools like Telnet, the raw GET command reveals how headers and payloads are actually handled. Because HTTP does not encrypt data, full sessions can be intercepted—including cookies and credentials—unless TLS is used (HTTPS).

---

## FTP – Cleartext Authentication and Dual Channels (Port 21)

I first attempted to connect via Telnet to observe the control channel:

```bash
telnet <target-ip> 21
```

After connecting:

```
USER frank
PASS D2xc9CgD
```

I then issued the following commands:

* `SYST` — Returned system type.
* `PASV` — Requested passive mode (where the client initiates both connections).
* `TYPE A` — Switched to ASCII transfer mode.

Because FTP uses a separate channel for file transfers, I switched to an actual FTP client:

```bash
ftp <target-ip>
```

After logging in:

* `ls` listed the files.
* `get ftp-flag.thm` downloaded the file.
* `cat ftp-flag.thm` displayed the contents.

**Flag**: `THM{364db6ad0e3ddfe7bf0b1870fb06fbdf}`

**Context**: FTP is often left open in legacy systems for file sharing, but its reliance on separate control and data channels—both of which are unauthenticated and unencrypted—makes it highly vulnerable to interception and manipulation. Passive mode is often required in modern environments due to firewall restrictions, but this adds complexity and increases attack surface.

---

## SMTP – Email Transmission Without Encryption (Port 25)

I connected to the SMTP server:

```bash
telnet <target-ip> 25
```

Then followed the SMTP flow:

```
HELO mydomain.com
MAIL FROM:<frank@mydomain.com>
RCPT TO:<admin@mydomain.com>
DATA
Test message for SMTP flag.
.
```

The server responded and queued the message. From the server response, I retrieved the flag:

**Flag**: `THM{5b31ddfc0c11d81eba776e983c35e9b5}`

**Context**: SMTP, in its default configuration, does not require encryption or authentication. This allows attackers to spoof messages or capture sensitive email content in transit. Secure mail relay setups typically implement STARTTLS or use SMTPS over port 465 to mitigate these risks.

---

## POP3 – Retrieving Mailbox Metadata (Port 110)

I connected via Telnet:

```bash
telnet <target-ip> 110
```

Then entered:

```
USER frank
PASS D2xc9CgD
STAT
```

This command returned:

**Response**: `+OK 0 0`

**Context**: POP3 is designed for single-client download access. It pulls messages from the server and optionally deletes them, making synchronization across devices difficult. In its raw form, all communication is exposed—including credentials—posing a significant threat in transit.

---

## IMAP – Synchronizing Mail Across Devices (Port 143)

I tested IMAP next:

```bash
telnet <target-ip> 143
```

Then sent:

```
a1 LOGIN frank D2xc9CgD
a2 LIST "" "*"
a3 EXAMINE INBOX
```

The server indicated there were no messages to retrieve.

**Answer**: `0`

**Context**: IMAP is more advanced than POP3, enabling multi-device access by keeping emails on the server and synchronizing states. However, like POP3, IMAP communicates in cleartext by default unless secured via STARTTLS or IMAPS on port 993. Its command-tagging system is useful for parsing responses in long-running sessions.

---

## Summary of Protocols and Ports

| Protocol | Default Port | Use Case                  | Secure Alternative |
| -------- | ------------ | ------------------------- | ------------------ |
| Telnet   | 23           | Remote terminal access    | SSH (port 22)      |
| HTTP     | 80           | Web page delivery         | HTTPS (port 443)   |
| FTP      | 21           | File transfer             | SFTP / FTPS        |
| SMTP     | 25           | Sending email             | SMTPS / STARTTLS   |
| POP3     | 110          | Retrieving email          | POP3S (port 995)   |
| IMAP     | 143          | Email sync across devices | IMAPS (port 993)   |

---

## Lessons Learned

### 1. Plaintext Protocols Still Exist in Production

Despite the known risks, many organizations continue to expose services like FTP, Telnet, and POP3 to the internet. These services may function correctly but introduce significant risk by making authentication data and content visible on the wire.

### 2. Encryption Must Be Enforced by Default

Even in internal networks, protocols that expose data in cleartext must be replaced with secure alternatives. Encryption should not be optional. It is now a baseline requirement under frameworks such as NIST 800-53, ISO 27001, and HIPAA.

### 3. Visibility into Protocol Behavior Is Critical

By manually interacting with each protocol, I was able to observe exactly what data was sent, how it was structured, and what was returned. This kind of visibility is crucial for tuning intrusion detection systems (IDS), creating accurate SIEM correlation rules, and performing incident response effectively.

### 4. Legal Risk Mirrors Technical Risk

Under laws like GDPR and CCPA, failing to encrypt personal data—including names and login credentials—can trigger liability in the event of a breach. Using outdated protocols without proper safeguards could result in enforcement actions, fines, or litigation.

### 5. Protocol Knowledge Enhances Defensive Strategy

Understanding how services like IMAP, SMTP, and FTP communicate enables more effective firewall rules, threat modeling, and configuration baselines. This is especially important when hardening servers or conducting internal security audits.
