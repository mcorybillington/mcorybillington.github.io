---
layout: default
title: OSINT
---
# Cool things on the internet...
## Google dorks
Dorks I have submitted to the [Google Hacking Database](https://www.exploit-db.com/google-hacking-database?author=10095):
### EdgeOS instances for Ubiquiti devices
`intitle:"EdgeOS" intext:"Please login"`  
[intitle:"EdgeOS" intext:"Please login"](https://www.google.com/search?q=intitle%3A%22EdgeOS%22+intext%3A%22Please+login%22)  
### Brocade switches
`intitle:"Web Management Login"`  
[intitle:"Web Management Login"](https://www.google.com/search?q=intitle%3A%22Web+Management+Login%22)  
### Various .pl login portals
`inurl:cgi/login.pl`  
[inurl:cgi/login.pl](https://www.google.com/search?q=inurl%3Acgi%2Flogin.pl)  
### Matrix Science database Instances
`inurl:cgi/login.pl intext:"Matrix Science"`  
[inurl:cgi/login.pl intext:"Matrix Science"](https://www.google.com/search?q=inurl%3Acgi%2Flogin.pl)   
### Web portals for Local Run Manager software from Illumina
`intitle:"Local Run Manager" intext:"Local Run Manager Version:"`  
[intitle:"Local Run Manager" intext:"Local Run Manager Version:"](https://www.google.com/search?q=intitle:%22Local%20Run%20Manager%22%20intext:%22Local%20Run%20Manager%20Version:%22)  
### Web portals for Ricoh printers/copiers/multifunction machines
`inurl:webArch/mainFrame filetype:cgi intext:"Web Image Monitor"`  
[inurl:webArch/mainFrame filetype:cgi intext:"Web Image Monitor"](https://www.google.com/search?q=inurl:webArch/mainFrame%20filetype:cgi%20intext:%22Web%20Image%20Monitor%22)    
### Magento Admin Panels
`intitle:"Magento Admin" intext:"Log in to Admin Panel"`  
[intitle:"Magento Admin" intext:"Log in to Admin Panel"](https://www.google.com/search?q=intitle%3A%22Magento+Admin%22+intext%3A%22Log+in+to+Admin+Panel%22&oq=intitle%3A%22Magento+Admin%22+intext%3A%22Log+in+to+Admin+Panel%22)  
## Shodan queries
### EdgeOS instances for Ubiquiti devices
`http.favicon.hash:-1677255344`  
[http.favicon.hash:-1677255344](https://www.shodan.io/search?query=http.favicon.hash%3A-1677255344)  
### Infoblox login portals
`http.favicon.hash:-195335685`  
[http.favicon.hash:-195335685](https://www.shodan.io/search?query=http.favicon.hash%3A-195335685)  
### Brocade switches
`http.title:"Web Management Login"`  
[http.title:"Web Management Login"](https://www.shodan.io/search?query=http.title%3A%22Web+Management+Login%22)  
### WebMin vulnerable to CVE-2019-15107
`MiniServ/1.890 OR MiniServ/1.900 OR MiniServ/1.910 OR MiniServ/1.920`  
[http.title:"Web Management Login"](https://www.shodan.io/search?query=MiniServ%2F1.890+OR+MiniServ%2F1.900+OR+MiniServ%2F1.910+OR+MiniServ%2F1.920)  
