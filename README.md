# Wireshark-and-Nmap-Packet-Analysis-and-Network-Reconnaissance

Hi everyone, welcome here to another repository where I talk about how I used **Wireshark** (a tool that lets you see every "letter" sent over the network) and **Nmap** (a tool used to scan open doors) in a recent lab on my Virtual machine. I moved from simple tests to finding a real security "incident" where a password was stolen because it wasn't protected. To be good at cybersecurity, you must know how computers talk to each other. If you can't see the traffic, you can't stop the hackers.

---

> [!IMPORTANT]
> **Lab Environment Transparency:** This analysis was performed in a controlled VirtualBox instance using **NAT networking**. I have intentionally avoided blurring internal IP addresses (10.0.2.x) to preserve the forensic integrity of the screenshots and provide an optimal viewing experience. Because these addresses are private and non-identifiable, they pose no security risk to the host, while allowing the audience to follow the packet flow accurately.
> 
> Please note that while internal IPs are private, external targets (such as Google’s Public DNS at 8.8.8.8) remain visible as standard public baselines used to verify outbound connectivity and ICMP response cycles.

---

### Objectives
This lab demonstrates how to use **Wireshark** and **Nmap** to monitor traffic and identify security risks in a virtual environment. I captured live data to spot network scans and documented a critical vulnerability where passwords were sent in plain text. This project highlights my ability to analyze packet-level data and customize tools for faster threat detection.

### Tools I Used:
* **Wireshark:** My main security camera for the network
* **Nmap:** Used to simulate an attacker's mission
* **Linux (Ubuntu):** The system where I ran all my commands
* **VirtualBox:** To run my virtual lab safely
* **Curl:** A quick way to send data to websites from the terminal

---

### 7 Phases of the Lab

**Phase 1-3:** I established a connection with a server and watched the TCP handshake to see how computers agree to talk before sharing data.

**Phase 4-6:** I looked at the background whispers on the network and used Nmap to scan for open doors. I then set up custom coloring rules to highlight those scans in bright red so they are easier to spot.

**Phase 7:** I found a high-risk security flaw where a username and password were sent over the network without any encryption. I showed how anyone watching the traffic could see this sensitive information in plain text.

---

### Phase 1: ICMP (The Ping Test)
I am going to capture a `ping` to see the basic request and response process. I started the capture by double-clicking on **enp0s3**. Then I opened the terminal and initiated a ping by typing `ping -c 4 8.8.8.8` which is **Google Public DNS server** operated by Google. It reliably responds to **ICMP Echo Requests**. When the `ping` finishes I stopped the capture and typed `icmp` at the display filter and applied it.

<img width="720" height="442" alt="image" src="https://github.com/user-attachments/assets/46ce50b9-2790-4031-98fa-420fec576c88" />


As we can see, I found 8 packets: 4 **Echo (ping) requests** and 4 **Echo (ping) replies**. I then clicked the first packet which is an Echo (ping) request. In the packet details, I expanded the **Internet Control Message Protocol** section where I found and marked **Type: 8 (Echo (ping) request)**.

Before looking for complex threats, an analyst must verify that the target is reachable. Testing with **ICMP (ping)** is the fastest way to confirm that the network path is open and the server is responding. This provides a baseline for all the security analysis that follows.

---

### Phase 2: DNS (Finding Addresses)
Now what I did for **DNS** here is exactly the same as what I did for ICMP. This shows the "question" my computer asked and the "answer" the server gave back.

Here I typed `nslookup google.com` and ran that on the terminal. Then I went back to Wireshark, stopped the capture, and applied the `dns` filter. Using this filter only shows DNS packets. I clicked on the top one and expanded the details.

<img width="720" height="410" alt="image" src="https://github.com/user-attachments/assets/376d7eaa-e597-4502-b3b1-6a984001d89e" />


Inside packet details I expanded the **Queries** where I see `Name: google.com, TYPE: A, and CLASS: IN`.

I clicked the next packet that came right after the one we worked on so far in Phase 2. In the middle pane **Answers** section I see the actual IP addresses that Google's servers sent back to my VM.

Every connection starts with a name. By analyzing **DNS traffic**, an analyst can see exactly where a system is trying to go before the connection even happens. This is important for security because attackers often use fake or malicious domain names. Monitoring these requests helps ensure that traffic is heading to the correct safe IP address and not a malicious site.

---

### Phase 3: TCP (The 3-Way Handshake)
To make this easy to find in Wireshark I used a website that does not use encryption (Plain HTTP). I started a capture and ran the `curl http://neverssl.com` command on the terminal. After that I came back to Wireshark to stop the capture and on the display filter I typed `tcp` and hit enter.

<img width="720" height="455" alt="image" src="https://github.com/user-attachments/assets/2ec5eab7-ca23-4a7d-b3f1-3373d33b600b" />


I found 3 packets in a row between my VM IP and the website IP which I am explaining below:

* **[SYN]** -> The "Synchronize" request from my VM
* **[SYN, ACK]** -> The server said to my VM "I hear you, let's talk"
* **[ACK]** -> My VM said "Got it, let's go"

I also clicked on the first packet and expanded the **Transmission Control Protocol** section. From there I expanded the **Flags** section and saw the line `Syn: Set`.

I also noticed packets 10 to 13 **[FIN, ACK]** which is a **4-way termination** where both sides politely agree on ending the conversation.

Reliable network communication depends on the **TCP handshake**. By isolating the **SYN**, **SYN-ACK**, and **ACK** packets I verified the formal "agreement" that establishes a session. This is a critical defensive skill because it allows an analyst to distinguish between successful connections and potential attacks like **SYN floods** that try to overwhelm a server by leaving handshakes unfinished.

---

### Phase 4: ARP (Network Background Noise)
Here I typed `curl http://neverssl.com` in my terminal and started the capture. I then applied `tcp` on display filter, after that I selected the first packet and from inside the packet details I expanded Transmission Control Protocol and from there I expanded Flags sub-section where I see a `1` next to `Syn` which is bit that is used by TCP headers  to communicate and tells Wireshark  this packet is request to start a new conversation.

<img width="720" height="423" alt="image" src="https://github.com/user-attachments/assets/224b7358-c0bb-414d-b51c-545c921da43a" />


In summary, I looked for `.... .... ..1. = Syn: Set`
* If it says `Set`, the bit is 1.
* If it says `Not Set`, the bit is 0.

Networks are never truly silent. Capturing **ARP** broadcasts reveals the background noises of normal administrative traffic. Mastering this allows an analyst to filter out irrelevant data and focus on anomalies that could indicate spoofing or unauthorized device discovery.

---

### Phase 5: Nmap (Active Reconnaissance)
In this phase I will go from **Network Basics** to **Security Analyst Reconnaissance**. In a real-world scenario, a SOC analyst would see this kind of traffic and immediately flag it as a potential scan or "pre-attack" activity.

Here I am going to run a **Service Version Scan** which is noisier than a basic ping. Nmap will try talking to multiple ports to see what software is running. I will use the following command in my terminal and start the capture in Wireshark:

```bash
nmap -sV 8.8.8.8
```

<img width="720" height="435" alt="image" src="https://github.com/user-attachments/assets/9b54c1aa-370a-413c-b817-905d9ba8f9e5" />

Here `-sV` tells Nmap to probe open ports to determine service/version info and `8.8.8.8` is Google’s DNS Server, which is a reliable and public target. After I ran the terminal, I saw a storm of packets appearing in Wireshark and I waited about a minute for Nmap to finish and stopped the capture. To see exactly what Nmap did I typed `tcp.flags.syn == 1 && tcp.flags.ack == 0` and this showed me every single connection request Nmap sent. We can notice how it hit many different ports (like 80, 443, and 22) in a very short time.

By running an **Nmap service version scan**, I simulated how an attacker probes a network to find open ports and software versions. Identifying this noisy traffic in **Wireshark** is a core skill for any analyst. It allows for the early detection of a potential breach before an attacker can exploit a specific vulnerability.

### Phase 6: Coloring Rules

Once I saw how many packets an Nmap scan creates, I realized it is hard to see what is important. In Phase 6, I created a rule to highlight these specific packets in bright red.

<img width="1125" height="986" alt="image" src="https://github.com/user-attachments/assets/7fdb00c4-82ec-4f10-a3ba-b4988edac732" />


**What I did here:**

1. Clicked on **View** → **Coloring Rules**.
2. Then I created a new rule by clicking the `+` button (check bottom left corner).
3. I named the rule `Active Reconnaissance` and set the filter as the one I typed in the search bar: `tcp.flags.syn == 1 && tcp.flags.ack == 0` (Marked in yellow colored box).
4. I picked a bright red as background with white as foreground color. This makes it much easier to spot an attacker in a crowded network.

<img width="1125" height="590" alt="image" src="https://github.com/user-attachments/assets/4430ea61-f076-4c09-96df-b897448142f6" />


In a real-world SOC environment, an analyst may face thousands of lines of traffic, making these "knocks on the door" nearly impossible to find manually. This phase demonstrates how to solve that challenge by increasing analyst efficiency through packet organization. By transforming the unorganized data from Phase 5 into a prioritized visual layout, I successfully reduced the **Mean Time to Detect (MTTD)**. This step is evidence of a professional workflow designed to cut through noise and identify threats in seconds.

### Phase 7: HTTP (The Stolen Password)

In the final phase of this lab, I simulated a high-severity security incident which is the transmission of sensitive credentials over an unencrypted channel. We are going to use `curl` to log in to a site that does not use encryption. I will start the capture and run the following command in my terminal:

```bash
curl -u "admin:SecurityPass123" http://testphp.vulnweb.com/login.php
```
This command sends the username `admin` and the password `SecurityPass123` over a standard, unencrypted HTTP connection.

<img width="1125" height="691" alt="image" src="https://github.com/user-attachments/assets/7b7f75e1-3fa6-41b5-802a-234a95e89605" />


Here I found exactly how a hacker steals a password. I typed `http` in the filter to find the login activity. I looked for a specific packet that said `GET /login.php` and when I clicked that packet and looked inside the **Hypertext Transfer Protocol** section I found a line called Authorization. Because the website was using an old connection called HTTP there was no encryption. I right-clicked that line and Wireshark showed me the **Credentials** in plain text as we can see highlighted in yellow.

Here I demonstrated how easily sensitive data can be intercepted via sniffing. This phase proves why encrypted protocols like **HTTPS** are a requirement as it highlights the real-world impact of a security vulnerability on user data and organizational integrity.

### Final Thoughts
This lab provided a hands-on look at the relationship between network reconnaissance and defensive analysis. Using Nmap helped me understand the specific "noise" an attacker makes when probing for vulnerabilities while **Wireshark** proved to be an essential tool for cutting through that noise to find real threats.

Building this environment in a **Virtual Machine** allowed me to safely simulate a high-risk scenario and see exactly how unencrypted protocols like **HTTP** can lead to a total credential breach. Beyond just capturing traffic I learned that customizing tool settings, like coloring rules, which is a requirement for any analyst who wants to work efficiently in a fast-paced environment. Mastering these fundamentals is a critical step in developing a proactive mindset for security operations.


### Connect & Explore

**GitHub**: Thank you for following along! This lab is part of my ongoing commitment to mastering incident response and forensic analysis. I’m constantly adding new labs and documentation—check back soon for upcoming projects! Check out my other works below: 
*  [**SQL-Based Security Incident Investigation**](https://github.com/safayetchowdhury/SQL-based-Security-Incident-Investigation)
*  [**Penetration Testing & Forensic Audit**](https://github.com/safayetchowdhury/Penetration-Testing-Forensic-Audit-Using-Kali-Ubuntu-Metasploit)
*  [**Defensive Architecture Using Snort IDS and SOAR**](https://github.com/safayetchowdhury/Defensive-Architecture-Using-Snort-IDS-and-SOAR)
*  [**Incident Response with Splunk**](https://github.com/safayetchowdhury/Incident-Response-with-Splunk)

**LinkedIn:** I’d love to **[connect](https://www.linkedin.com/in/chowdhurysafayet/)**! If you’re working in information security or are interested in these methodologies, let's discuss how we can improve security posture together.
