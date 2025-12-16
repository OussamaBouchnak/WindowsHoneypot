# WindowsHoneypot

A cybersecurity learning project that demonstrates how to set up a Windows-based honeypot using Azure Virtual Machines to attract and analyze cyberattacks.

## What is a Honeypot?

A honeypot is a sacrificial computer system designed to attract cyberattacks, acting as a decoy. It mimics a target for hackers and uses their intrusion attempts to gain information about cybercriminals and their methods.

**Source:** [Kaspersky - What is a Honeypot](https://www.kaspersky.com/resource-center/threats/what-is-a-honeypot)

## Project Overview

This honeypot is an intentionally unprotected Windows VM connected to the internet that will attract attackers. We then collect and analyze the attack data using Azure Sentinel (SIEM) to track attacker behavior and learn about real-world cyber threats.

## Architecture

<img width="995" height="478" alt="image" src="https://github.com/user-attachments/assets/0fe037c3-e410-4159-a112-0c9b0b5317a0" />

## Setup Steps

### 1. Create Azure Resource Group

A resource group is a logical container for Azure resources. All resources in this project will be organized under one resource group.

* Navigate to Azure Portal
* Create a new Resource Group
* Choose our preferred region

### 2. Create Virtual Network

A virtual network provides the network infrastructure for our VM to connect to the internet.

* Create a Virtual Network in our resource group
* Use default parameters
* Note the address range assigned (this is where our VM's IP will come from)

### 3. Deploy Windows Virtual Machine

* Create a Windows VM in Azure
* Connect it to the virtual network created in step 2
* Note the public IP address for remote access

### 4. Configure Network Security (Make it Vulnerable)

**This will make our VM vulnerable to attacks.**

* Open Network Security Group (NSG) settings
* Add inbound security rule to allow ALL traffic:

  * Protocol: Any
  * Source: Any (*)
  * Destination: Any (*)
  * Action: Allow

### 5. Disable Windows Firewall

Connect to our VM via Remote Desktop Connection (RDP) using the public IP address, then:

* Open Windows Defender Firewall settings
* Turn off firewall for all network profiles (Domain, Private, Public)

### 6. Verify VM is Exposed

From our local machine, ping the VM's public IP address to confirm it's reachable from the internet. <img width="954" height="331" alt="image" src="https://github.com/user-attachments/assets/8279c0b2-137b-43d4-9c72-f3a1b920ed2d" />

### 7. Set Up Log Collection

**Create Log Analytics Workspace:**

* Navigate to Azure Portal
* Create a Log Analytics Workspace
* This will serve as our log repository

  <img width="954" height="602" alt="image" src="https://github.com/user-attachments/assets/2f9db941-ff9e-433b-a668-18b2a844d9ba" />

**Deploy Azure Sentinel:**

* Create Azure Sentinel instance
* Connect it to our Log Analytics Workspace

  <img width="1014" height="650" alt="image" src="https://github.com/user-attachments/assets/88418fe9-4302-4480-b113-a52a5fcdf40a" />

### 8. Connect VM to Logging Infrastructure

**Install Windows Security Events Connector:**

* Go to Sentinel → Content Management → Content Hub
* Install "Windows Security Events" solution
* Open the connector page

  <img width="1975" height="997" alt="image" src="https://github.com/user-attachments/assets/f67755a8-98b5-4878-beae-9568b7056687" />

**Create Data Collection Rule:**

* Navigate to "Windows Security Events via AMA"
* Create a new Data Collection Rule (DCR)
* Link our honeypot VM to this rule
* This forwards all security logs from our VM to Log Analytics

  <img width="997" height="426" alt="image" src="https://github.com/user-attachments/assets/c29bd39e-b053-4b82-ae68-85e17846374f" />

### 9. Analyze Logs with KQL

Once logs start flowing in (may take a few minutes), we can query them using Kusto Query Language (KQL):

```kql
SecurityEvent
| where TimeGenerated > ago(24h)
| take 100
```

The `SecurityEvent` table contains all security events forwarded by the Azure Monitor Agent from our honeypot.

## Future Enhancements

* Geolocation mapping of attackers
* Custom alert rules for specific attack patterns
* Automated threat intelligence integration
* Dashboard creation for visualization


