# 🔍 TryHackMe — Tshark Challenge 1: Teamwork | SOC Investigation Writeup

> **Platform**: TryHackMe<br>
> **Room**: Tsharl Challenge 1 : teamwork <br>
> **Focus**: Tshark Analysis network log

---

## 🎯 Scenario

In this scenario an alert has been trigerred "The threat research team discovered a suspicious domain that could be a potential threat to the organisation." 
We only have this information and the file .pcap in the Desktop. And the objectif of that room is to pratice ours Tshark Skill. That perfect  because lot of time we are focused with the GIU , but its still very important 
To work with CLi or GUI ! 

---

### Identifying the malicious Domain 


First i need to discover the malicious domain, i need an overview of the web traffic, and my first mind is to look HTTP Get request/ 


tshark -r teamwork.pcap -Y "http.request.method == GET"

I discover reveals multiple request to an external IP Address 184.154.127.226 fetching files like /suspecious.php, /js/cc.js , paypalsansmallmedium.woff2 



