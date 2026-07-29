# Phase 1 — Building the Network
 
Goal: Bloom & Brew's web server needs somewhere to live. Before launching any server, I built the network around it from scratch: an isolated VPC, a public subnet, a path to the internet, and a firewall. AWS gives you a default VPC that would have worked fine, but I skipped it on purpose. Building each piece by hand forced me to understand what each one actually does.
 
## Architecture at the end of this phase
 
```
Internet
   |
[Internet Gateway: bloombrew-igw]
   |
VPC: bloombrew-vpc (10.0.0.0/16)
   |
Public Subnet: bloombrew-public-subnet (10.0.1.0/24, us-east-1a)
   - route table: bloombrew-public-rt (0.0.0.0/0 -> IGW)
   - firewall: bloombrew-web-sg (HTTP 80: world / SSH 22: my IP only)
```
 
## 1. VPC — bloombrew-vpc
 
Created a VPC with CIDR block 10.0.0.0/16, which gives me about 65,000 private addresses from the RFC 1918 range. This is the isolated network boundary that everything else lives inside. Nothing gets in or out unless I build a path for it.
 
I left "VPC encryption control" off. It's a paid feature (AWS marks it with a `($)` next to the name) and this project doesn't need it. Good habit I picked up on day one: whenever AWS puts a `($)` next to something, stop and ask if you actually need it.
 
![VPC details](images/vpc-details.png)
 
## 2. Subnet — bloombrew-public-subnet
 
Carved a 10.0.1.0/24 subnet (256 addresses) out of the VPC's range, in availability zone us-east-1a.
 
The create screen confused me for a minute. There are two nearly identically named fields stacked on top of each other: "IPv4 VPC CIDR block" (a dropdown showing the parent /16) and "IPv4 subnet CIDR block" (where my /24 goes). The subnet has to fit inside the VPC's range, so AWS shows you the parent and asks for the slice. Once I understood that, the whole parent/slice relationship clicked.
 
![Subnet details with route table](images/subnet-details.png)
 
## 3. Internet Gateway — bloombrew-igw
 
Created an internet gateway and attached it to the VPC. The IGW is basically the door between the VPC and the internet. Attaching it installs the door, but no traffic flows until a route points at it. That's the next step.
 
Honest note: I originally named this thing bloombrew-public-rt, which is the route table's name, because I was creating several objects back to back and crossed my wires. I only noticed when the page title looked wrong. Took 10 seconds to fix in Manage tags, since AWS names are really just editable tags. Still, lesson learned about naming discipline when you're making a bunch of resources in a row.
 
![Internet gateway attached](images/igw-attached.png)
 
## 4. Route Table — bloombrew-public-rt
 
Created a route table in the VPC and added one route: destination 0.0.0.0/0 (meaning every address anywhere) with the internet gateway as the target. Then associated the route table with my subnet.
 
The table ends up with two routes:
 
| Destination | Target | Meaning |
|---|---|---|
| 10.0.0.0/16 | local | traffic inside the VPC (added automatically, can't be removed) |
| 0.0.0.0/0 | igw-… | everything else goes out to the internet |
 
This step answers the classic interview question "what makes a subnet public": a subnet is public when its route table has a route to an internet gateway. I got to watch it happen. Before I saved the association, the console showed my subnet on the VPC's default "Main" route table, which has no internet route. After saving, it pointed at mine.
 
Extra honest note: the first time around, I checked the box on the association screen but never actually clicked Save. I only caught it later while taking screenshots for this write-up, when the associations page said "Explicit subnet associations (0)". So my "public" subnet spent a few hours not being public at all. If I had launched the server before noticing, the site would have been unreachable and I would have spent a while hunting for the reason. Turns out documenting your work is also how you verify it.
 
![Route table routes](images/route-table-routes.png)
 
![Route table subnet associations](images/route-table-associations.png)
 
## 5. Security Group — bloombrew-web-sg
 
The instance-level firewall. Two inbound rules:
 
| Port | Protocol | Source | Why |
|---|---|---|---|
| 80 (HTTP) | TCP | 0.0.0.0/0 | it's a public website, the world is supposed to reach it |
| 22 (SSH) | TCP | my IP /32 | server administration is just for me |
 
Outbound is the default allow-all.
 
AWS put up its yellow warning about 0.0.0.0/0 allowing all IP addresses. For port 80 that's fine, since an open-to-everyone web port is the whole point of a public website. The same source on port 22 would be a genuine problem, which is why SSH is pinned to a /32, meaning exactly one address: mine.
 
Things I learned on this screen. Security groups are stateful: if a request is allowed in, the reply is automatically allowed out, no reply rules needed. They're also deny-by-default, so any port I didn't open is closed. And the create form has three different description fields, one required for the whole group plus an optional one on each rule. I found this out by filling in the wrong one first.
 
![Security group inbound rules](images/security-group-rules.png)
 
## CIDR sizes I touched today
 
/16 for the whole VPC (~65k addresses), /24 for one subnet (256), /32 for exactly one machine (my IP in the SSH rule). Same notation at three different scopes. Seeing all three in one afternoon is what finally made CIDR stick for me.
 
## What's next
 
Phase 2: launch an EC2 instance into this subnet, install Nginx, and get Bloom & Brew's site on the internet.
