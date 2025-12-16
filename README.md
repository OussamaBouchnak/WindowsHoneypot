# WindowsHoneypot

A cybersecurity learning project that demonstrates how to set up a Windows-based honeypot using Azure Virtual Machines to attract and analyze cyberattacks.
<img width="1452" height="805" alt="image" src="https://github.com/user-attachments/assets/3c8fd4cb-f541-4017-a84a-1a88fea1b315" />

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

### 2. Create Virtual Network

A virtual network provides the network infrastructure for our VM to connect to the internet.

### 3. Deploy Windows Virtual Machine

* Create a Windows VM in Azure
* Connect it to the virtual network created in step 2

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


**Create Data Collection Rule:**

* Navigate to "Windows Security Events via AMA" (Azure Monitoring Agent)
* Create a new Data Collection Rule (DCR)
* Link our honeypot VM to this rule
* This forwards all security logs from our VM to Log Analytics
<img width="997" height="426" alt="image" src="https://github.com/user-attachments/assets/c29bd39e-b053-4b82-ae68-85e17846374f" />

### 9. Analyze Logs with KQL

Once logs start flowing in (may take a few minutes), we can query them using Kusto Query Language (KQL).
The `SecurityEvent` table contains all security events forwarded by the Azure Monitor Agent from our honeypot.
**4625** is the EventId for a failed logon attempt.

```kql
SecurityEvent
| where EventId == 4625
```

<img width="1190" height="743" alt="image" src="https://github.com/user-attachments/assets/2ddc7ac2-97f3-430e-aeb9-7d42589376f1" />

Within **less than one hour** of deployment, the honeypot received **+60,000 failed authentication attempts** from multiple external sources.

<img width="722" height="384" alt="image" src="https://github.com/user-attachments/assets/3ceb4136-99f4-4e1e-8071-e12896caf7e8" />

**Number of unique attacker IP addresses** attempting to authenticate against the honeypot:

<img width="870" height="307" alt="image" src="https://github.com/user-attachments/assets/8700a816-a4ea-4242-b537-f1eb2cc13b6e" />

**Most active attacker IP addresses:**

<img width="800" height="787" alt="image" src="https://github.com/user-attachments/assets/32f10e45-9833-4d8c-8f32-1d6c58aa5a05" />

This shows that a limited subset of sources was responsible for the majority of failed authentication attempts.
→ **Brute-force activity.**

**Most frequently targeted usernames**, typically default or administrative accounts:

<img width="680" height="444" alt="image" src="https://github.com/user-attachments/assets/f039b72c-d6e3-489d-9a0e-2a84c01f4886" />

This confirms the use of automated brute-force and dictionary attack tools.


## Geolocation Analysis of Attackers

### 10. Import Geolocation Data

To visualize where attacks are originating from, we need to map IP addresses to geographic locations.

**Obtain GeoIP Database:**

* Download a CSV file containing IP address ranges mapped to geographic locations
* Source: [Josh Makador](https://www.youtube.com/c/JoshMadakor) (highly recommended)

**Create Watchlist in Sentinel:**

* Navigate to Sentinel → Configuration → Watchlists
* Create a new watchlist:
  * Upload the geoip CSV file as the source
  * Set **Search Key** to `network`
  * Name the watchlist `geoip`

<img width="1403" height="820" alt="image" src="https://github.com/user-attachments/assets/c4153f61-5fca-4c23-a80c-194aaedc8a04" />

Once created, the watchlist will appear in our Sentinel configuration:

<img width="2095" height="852" alt="image" src="https://github.com/user-attachments/assets/bc7030d4-face-4d8b-9f8c-6db11cf3481b" />

### 11. Enrich Log Data with Geolocation

Return to Log Analytics Workspace to query the enriched data.

**The following KQL query** will correlate failed login attempts with geographic locations:

```kql
let GeoIPDB_FULL = _GetWatchlist("geoip");
let WindowsEvents = SecurityEvent
    | where EventID == 4625
    | order by TimeGenerated desc
    | evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network);
WindowsEvents
| project TimeGenerated, Computer, AttackerIp = IpAddress, cityname, countryname, latitude, longitude
```

This query:
* Loads the geoip watchlist
* Filters for failed authentication events (EventID 4625)
* Performs IP lookup to match attacker IPs with geographic data
* Projects only the relevant fields: timestamp, computer name, attacker IP, city, country, and coordinates

<img width="1551" height="1156" alt="image" src="https://github.com/user-attachments/assets/e8193647-7d3f-4b9b-8ff7-94d5732e5a91" />

### 12. Create Interactive Attack Map

Now we'll visualize the attack data on a world map to see where threats are coming from in real-time.

**Create Custom Workbook:**

* Navigate to Sentinel → Workbooks
* Click **Add workbook** → **Edit**
* Remove any existing elements
* Add Query element

**Configure Map Visualization:**

* Switch to **Advanced Editor**
* Paste the following JSON configuration:

```json
{
	"type": 3,
	"content": {
	"version": "KqlItem/1.0",
	"query": "let GeoIPDB_FULL = _GetWatchlist(\"geoip\");\nlet WindowsEvents = SecurityEvent;\nWindowsEvents | where EventID == 4625\n| order by TimeGenerated desc\n| evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network)\n| summarize FailureCount = count() by IpAddress, latitude, longitude, cityname, countryname\n| project FailureCount, AttackerIp = IpAddress, latitude, longitude, city = cityname, country = countryname,\nfriendly_location = strcat(cityname, \" (\", countryname, \")\");",
	"size": 3,
	"timeContext": {
		"durationMs": 2592000000
	},
	"queryType": 0,
	"resourceType": "microsoft.operationalinsights/workspaces",
	"visualization": "map",
	"mapSettings": {
		"locInfo": "LatLong",
		"locInfoColumn": "countryname",
		"latitude": "latitude",
		"longitude": "longitude",
		"sizeSettings": "FailureCount",
		"sizeAggregation": "Sum",
		"opacity": 0.8,
		"labelSettings": "friendly_location",
		"legendMetric": "FailureCount",
		"legendAggregation": "Sum",
		"itemColorSettings": {
		"nodeColorField": "FailureCount",
		"colorAggregation": "Sum",
		"type": "heatmap",
		"heatmapPalette": "greenRed"
		}
	}
	},
	"name": "query - 0"
}
```

* Save the workbook

<img width="1452" height="805" alt="image" src="https://github.com/user-attachments/assets/3c8fd4cb-f541-4017-a84a-1a88fea1b315" />

**Result:** An interactive heat map showing:
* Attack origins by geographic location
* Attack intensity indicated by color (green to red)
* Bubble size representing the number of failed authentication attempts per location
* Real-time updates as new attacks occur

The map will continue to update in real-time as long as the honeypot VM remains active.

## Future Enhancements
* Custom alert rules for specific attack patterns
* Automated threat intelligence integration
* Dashboard creation for visualization
