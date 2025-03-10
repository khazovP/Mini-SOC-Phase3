**Phase 3: Implementing EDR, Threat Intelligence, and Automation**

### Introduction
With the foundational setup completed in the previous phases, it was time to take my **Azure Mini SOC** to the next level. In this phase, I aimed to implement an **Endpoint Detection and Response (EDR) solution**, send alerts to **Microsoft Sentinel**, integrate **Threat Intelligence**, and install an **EDR agent** on an endpoint to simulate attacks. Additionally, I wanted to explore **security automation** by automatically blocking threats based on alerts.
![Topology v2 Sentinel Automation](https://github.com/user-attachments/assets/e39805b6-80c9-45d2-bf5c-6ed2d334ff93)

---

## **Deploying Wazuh EDR**

### **Choosing the EDR Solution**
After evaluating different EDR solutions, I decided to go with **Wazuh**—an open-source, free alternative that provides SIEM and XDR functionalities. The idea was simple: **monitor endpoints, detect threats, and forward critical alerts to Sentinel**.

### **Installation Troubles**
I started by downloading Wazuh from the official website and attempted to install it on my **Debian-based** VM. **Big mistake.** The installation failed miserably. After digging through the documentation, I found out that **Debian is not officially supported**. However, I wasn’t ready to give up just yet—I tried installing it with the `--ignore-hardware-requirements` flag, hoping it would magically work. **Spoiler alert: It didn't.**
![image](https://github.com/user-attachments/assets/54279199-39cb-48f6-9ec2-6a83d166475c)

At this point, I had two choices:
1. **Try troubleshooting** and force an installation on Debian (probably a bad idea)
2. **Recreate the VM using a supported OS (Ubuntu)**

I went with the second option, hoping that the failure was due to software compatibility and not hardware limitations.

### **Deploying a New Ubuntu VM**
To ensure a clean setup, I:
- Went to **Sentinel → Data Connectors** and detached the failing VM
![image](https://github.com/user-attachments/assets/64931d6f-1075-4d72-81f4-d0705ae601dc)

- Waited until the **Azure Monitor Agent stopped sending heartbeats** (optional but good practice)
![image](https://github.com/user-attachments/assets/2441e76f-09eb-46a2-8b22-0aac8e1a08e0)

- Removed the Debian VM and deployed a new one running **Ubuntu**

I logged into the new machine, ready to install Wazuh. **But then… another failure.** The installation still wouldn’t proceed. I dug into the logs and quickly realized the problem: the VM only had 1GB of RAM. No wonder the Java-based Wazuh manager wasn’t launching! I resized the VM to Standard DS1 v2 (1 vCPU, 3.5GB RAM), retried the installation and — success!
![image](https://github.com/user-attachments/assets/3cf8b7ac-d6ae-4f8b-8bef-f716ba328e1a)

#### **Firewall Configuration and Web UI Setup**
Before accessing the **Wazuh Web UI**, I had to create a **firewall policy** to allow traffic.
- Configured the FW to **permit access to the Wazuh Web UI**
![image](https://github.com/user-attachments/assets/1acdcd20-4571-4edd-8e2d-cee3e9460383)

- Used the credentials from the installation log to log in
- Web UI was now accessible
![image](https://github.com/user-attachments/assets/23484ee3-5e23-483a-892d-0e449b73e464)

To restore logging capabilities for this machine, I installed and configured **rsyslog** (as done in Phase 2). After a quick setup, **Palo Alto logs started flowing in**.
![image](https://github.com/user-attachments/assets/aada6877-af2d-4cdb-a55b-9d87164d07f8)

---

## **Deploying the Wazuh Agent**
Now that the Wazuh server was up and running, I needed to install an **agent** on an endpoint. Here’s how it went:

1. **Downloaded the Wazuh Agent**
   - Went to the **Wazuh dashboard** → **Deploy Agent**
   - Selected **Debian/Ubuntu (amd64)** package
   - Entered the **Wazuh server’s IP** and copied the installation command
![image](https://github.com/user-attachments/assets/0a4bc57d-ed36-4c79-921b-5f5a4daf9f5d)

2. **Installed and configured the agent**
   - Pasted the command on the endpoint
   - Reloaded the daemon list, enabled auto-startup, and started the agent
![image](https://github.com/user-attachments/assets/12f5576f-f972-4508-9aea-abc2e8a916cb)

3. **Monitored network traffic to identify agent communication**
   - Wazuh Agent was communicating over **ports 1515 and 1514**
![image](https://github.com/user-attachments/assets/d8ec09e1-517c-4d2f-81bc-b9dd99fb2ff3)

   - Created firewall policies to allow this traffic
![image](https://github.com/user-attachments/assets/be44a6cb-2a86-4778-8803-0e3a8853dc63)


4. **Confirmed successful deployment**
   - Went to **Wazuh Dashboard → Agent Management → Summary**
   - The agent was successfully checking in! ✅
![image](https://github.com/user-attachments/assets/7c93d085-644d-4b1e-9e96-206521f1090a)

---

## **Integrating Wazuh Alerts with Sentinel**
I decided that sending **all logs** from Wazuh to Sentinel would be overkill, so I configured **only alerts** to be forwarded.

### **Configuring Alert Forwarding**
1. Edited **`/var/ossec/etc/ossec.conf`** to forward alerts via **syslog in CEF format**
![image](https://github.com/user-attachments/assets/50ba5fc1-3c8a-40e6-9cb7-9950ced27f49)

2. Restarted Wazuh Manager: `systemctl restart wazuh-manager`
3. Simulated an attack to see if alerts were logged
To test, I used **Hydra** (brute-force tool) on **Kali Linux (WSL)** with custom wordlist to brute-force SSH.
![image](https://github.com/user-attachments/assets/98a1fb6a-17c7-446e-8d95-c965801bb0a1)

- Wazuh detected the attack ✅
- Alerts appeared in **syslog** ✅
![image](https://github.com/user-attachments/assets/501d02d0-ba6d-489b-a7d3-62e3a65c1e0d)

- Ai this point I recalled, that I have not recconnected DCR to new Machine. So, I wen to Sentinel Data Connectors and connected new Ubuntu Machine
![image](https://github.com/user-attachments/assets/01b4c3cc-8e3e-4f52-ab88-bdb54cef4b18)

- And after some time, we can see alerts arriving ✅
![image](https://github.com/user-attachments/assets/319579bd-a478-409f-a517-c1be560acbad)

---

## **Creating an Automation Rule in Sentinel**
Just receiving alerts wasn’t enough—I wanted **Sentinel to take action**. The idea was:
1. **Detect an attack** via Sentinel Analytics Rules
2. **Trigger an automated response** - in our case run a playbook
3. **Update the PA firewall's** predefined address group with new IP address

### **Step 1: Creating an Analytics Rule**
1. **New Alert Rule** → Named it appropriately
2. Set severity level
3. Configured query and timeframe. pay attention to "Alert Enchancement" section - it's not seen on screenshot, but I configured Entity mapping to extract source IP from logs, so it can be passed to automation rule later.
![6 1 wazuh alert](https://github.com/user-attachments/assets/f06668eb-819f-4b3f-ab93-1d019570d399)

4. Configure incident settings
![6 3 wazuh alert](https://github.com/user-attachments/assets/1f0dc462-78b4-47f2-8b5d-d7c9c44e8f98)

Now that our rule is created, it’s time to test it. To do so, I launched another brute-force attack using Hydra. 
![6 5 brute force](https://github.com/user-attachments/assets/4102ff47-887f-4333-832a-98535a8fd4f6)

After a short while, I checked Sentinel, and—boom!—an incident was automatically created.
![6 10](https://github.com/user-attachments/assets/9b0d72b1-08e0-4996-a1cd-3b4c47efe3d3)

At this point, Sentinel allows us to dive deeper into the incident. We can investigate it, view the timeline of events, analyze entities etc.
![6 11](https://github.com/user-attachments/assets/4a2d017c-f42a-4886-94fb-924c46bab9f8)
Sentinel can even autpmatically correlate logs for us.
![6 12 corelation](https://github.com/user-attachments/assets/ce840572-46a2-492e-85bd-c82c31dd5489)

---

### Setting Up the Playbook for Firewall Automation

#### Getting the Firewall API Key
First things first—I needed to get my firewall’s API key to allow Sentinel to interact with it. The key is obtained by issuing the following command:

```bash
curl -k -X GET "https://<FW_IP>/api/?type=keygen&user=<admin>&password=<password>"
```
Curl refused to connect to a server with self signed certificate, so I used "--insecure" flag.
[7 4 palo api](https://github.com/user-attachments/assets/302ebed0-52f1-4fff-97d5-2fd644d5e932)

The response contains an API key, which we will later use in the automation script.

Also, I created a special Address Group caleed "Blacklist" that will be populated with blocked IPs. For blocking two security policies are used - one with Blacklist source, second - with Blacklist in destination.
![7 9 policy mgmt](https://github.com/user-attachments/assets/4640227d-0b81-4605-bccd-5327bab6dc9a)

#### Deploying the Palo Alto Sentinel Connector

Microsoft provides an official Palo Alto connector for Sentinel, available in their GitHub repository:

🔗 [Azure Sentinel Palo Alto Connector](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/PaloAlto-PAN-OS)

I clicked "Deploy to Azure," filled in my firewall’s IP, and deployed the connector. But remember to fill in FW public IP, not private one, as Azure will connect from publicly facing services, not from inside of our network. I made a mistake at first attempts and had to re-deploy the connector.
![image](https://github.com/user-attachments/assets/1731b790-55de-44cb-b1ce-0021461e8b12)

At this moment I ran out of free credits and had to set up a new account on Azure. I recreated the same network, but without VPN (because it appeared to be the most cost eating service) and without all other details, because I have demonstrated it in previous phases.
![12](https://github.com/user-attachments/assets/2e32b588-efca-47db-977a-2db008b20ba3)

#### Deploying playbook
Next, I needed to set up actual automation rule. Go to Sentinel - Content Hub and install "PaloAlto-PAN-OS-BlockIP" playbook. After installing click "Create playbook".
![7 1 palo playbook](https://github.com/user-attachments/assets/bec1bce8-688a-4b77-a70f-ef44d3658b02)

Fill in the Address Group name (that will be used to block IPs) and some dummy string in Teams Group and channel ID.
![7 9 playbook](https://github.com/user-attachments/assets/6b6a29e5-dc29-43d7-9dca-145d1518901f)

Then, go to newly deployed "Paloaltoconnector-PaloAlto-PAN-OS-BlockIP" API Connection (you can find in Resource Group) and fill in the API key that we got from FW.
![7 11 edit api connection](https://github.com/user-attachments/assets/30c24c2d-76c1-4742-8b28-6a336e78c85d)


#### Encountering SSL/TLS Issues

When I attempted to run the playbook first time, I encountered another error:
![14](https://github.com/user-attachments/assets/e9cb19b4-532e-4ce6-8071-430cf336f7aa)

❌ **Azure refused to establish connection to FW**

After some research, I found this Microsoft documentation explaining that Logic Apps (which power Sentinel playbooks) require a **trusted SSL certificate**:

🔗 [Azure Logic Apps: SSL/TLS Requirements](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app?tabs=azure-portal#access-for-outbound-calls-to-other-services-and-systems)

Since my firewall was using a **self-signed cert**, Azure rejected the connection. 

#### Getting a Trusted SSL Certificate
I solved this problem in the following way: 
1. Buying a cheap domain from Namecheap.
3. Requesting an SSL certificate from **ZeroSSL** and passed domain ownership challenge.
5. Proving domain ownership by adding a **CNAME record** to my DNS.
![17](https://github.com/user-attachments/assets/06813de5-9142-4802-b948-19bfdd0dedee)

7. Adding an **A record** pointing to my firewall’s public IP.
![15](https://github.com/user-attachments/assets/1b700484-1218-4b17-87ce-4bb03b238170)

Once my certificate was issued, I imported it into the Palo Alto firewall:
- **Device > Certificate Management > Certificates** → Imported the new cert.
![20](https://github.com/user-attachments/assets/aade0018-3f8e-4134-9ae9-1721cca5f6ef)

- **SSL/TLS Service Profile** → Created a new profile using the cert.
![21](https://github.com/user-attachments/assets/931f6076-72f1-4673-9887-dc0b0e800e97)

- **Management > General Settings** → Applied the new profile.
![22](https://github.com/user-attachments/assets/56a94349-6af9-496d-89df-e492f6b9f2ee)

- **Commit Changes.**

After the firewall rebooted, I could now access its web UI via **HTTPS and my domain name**. More importantly, **Azure was finally able to connect! 🎉**
![image](https://github.com/user-attachments/assets/d37bc352-bb68-4025-b36c-8e88e6007c54)

#### Encountering Next Issue
Now that the SSL/TLS issue was resolved, I expected the playbook to work flawlessly. But nope. Another issue popped up.😅

Microsoft’s Palo Alto playbook **requires a working Teams Group ID and Channel ID** as Teams integration is mandatory in this playbook. The automation logic was designed to **send a Teams message**, asking whether to block or allow an IP. But I didn't have a paid Teams subscription.
![28](https://github.com/user-attachments/assets/aee23aff-08d6-4127-b821-2b7cf018ad35)

I searched for workarounds:
- I **tried signing up for a personal Teams account** — but it didn’t support creating "Teams".
- I **Googled how to get Teams API credentials**—no luck.
- I **discovered a free trial for Microsoft 365 Business**—but too late!😅

#### Hacking the Playbook to Remove Teams Logic
Since I had no business Teams account, I had to **modify the playbook** to remove the Teams approval step. After multiple failed attempts and errors, I finally stripped it down to its core functionality—**blocking an IP directly** without user approval.
![30](https://github.com/user-attachments/assets/13a67907-1be0-4600-9e9e-b0d0cf83e547)

🔧 **Final Adjustments:**
- Removed the "Post to Teams" logic.
- Edited the workflow to send the IP **directly to the firewall API**.
- Ensured Sentinel was correctly passing the entity (attacker's IP).

#### Testing the Automation

Now it was time for the final test! I launched another **brute-force attack**, triggered a Sentinel alert, and watched the automation in action. 

🎯 **Success!** The firewall’s **Blacklist Address Group was updated with the attacker’s IP**, and the FW automatically blocked incoming traffic from that IP.
![31](https://github.com/user-attachments/assets/708ddf8c-9be6-4937-8f9a-4e2b09493baf)

📸 Updated Address objects
![35](https://github.com/user-attachments/assets/105b1eeb-dfc0-4d85-8424-de3b085174bb)
![36](https://github.com/user-attachments/assets/dca35f50-d465-4d1f-9767-9727512ecea0)

#### Improving the Detection Logic
One last issue: Wazuh could only see post-NAT IP (the private IP of FW interface). This meant that **the real attacker’s IP was missing** from my automation. 

To fix this, with the help of AI, I wrote a **correlation query** in Sentinel to extract the **pre-NAT IP** from firewall logs, you can see it in column "Public IP":
![37 correlation](https://github.com/user-attachments/assets/95f425dd-2aad-4bf7-a218-2fcf8fc7a990)

I updated my **Analytics Rule** to use this query, and changed Entity mapping to use "Public IP" column, ensuring the correct attacker IP was being passed to my playbook.
![38 new rule logic](https://github.com/user-attachments/assets/cd68a3df-8b55-4d55-9865-56eced0ca3e7)

🔬 **Final Test:**
- **Launched an attack** → Sentinel alert triggered
![40 incident trigger](https://github.com/user-attachments/assets/368c2bd5-79b3-41a1-8bed-20d6a18f9b7e)

- **Sentinel create incident and mapped IP**
![40 incident trigger](https://github.com/user-attachments/assets/ac3fa2ea-99f9-4b1f-93fd-fbeb4e8281dd)

- **Playbook executed** with real attacker’s IP added to Blacklist object on FW!
![42 object updated](https://github.com/user-attachments/assets/51d9fc3b-b349-46db-b1b8-8a7f5cc18b4a)

🚀 **Mission accomplished!**

---

### Conclusion

This phase was a **huge learning experience**. I started with a simple goal—integrate Wazuh with Sentinel and automate blocking threats—but I faced many unexpected challenges along the way:
✅ Choosing an EDR solution and configuring alerts.
✅ Debugging Wazuh installation failures (first on Debian, then on Ubuntu).
✅ Troubleshooting SSL/TLS errors for API connectivity.
✅ Struggling with Microsoft Teams requirements and hacking the playbook.
✅ Enhancing detection logic to extract the correct IP.

In the end, **automation worked** exactly as intended. This was my first time implementing a fully automated security response, and despite the challenges, it was an incredibly rewarding experience. I did not plan Project any further, so probably this is the last part. Thank you if you have read it so far!🔥

