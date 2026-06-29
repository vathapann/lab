# A Reflection on My LANs & Routing Course at UTS

*The moment I finally understood how routers think*

## Opening reflection: why I took the course

I started my master's at UTS in Australia in the spring of 2025. After finishing my bachelor's and a few years working as a full-stack developer, my goal was to go deeper into the IT industry, and I was genuinely excited to be a student again for the first time in four years. The first semester took some adjusting: four courses here, compared to the six I was used to back home. One of them was LANs & Routing, a core subject all about networking and the internet. It turned out to be one of my favourites. Coming from full-stack work, I was curious about what actually happens underneath the apps I'd spent years building, and this course put me right in the middle of it.

### Core Learning concepts: VLANs, OSPF, NAT

### VLAN (Virtual Local Area Network)

I'd heard this term thrown around a lot during my part-time freelance work at an IT solutions company. Back then I didn't really understand it that much, just from watching, I figured it was a way to split one network into separate smaller networks, where the devices in each group could talk to each other. It turns out I wasn't far off.

![Photo of on-site part-time freelance work](https://raw.githubusercontent.com/vathapann/lab/main/blog-posts/how-router-really-think/assets/20211104_092718.jpg)

Here's the everyday version. Imagine an office with one big open floor and everyone working in the same room. A VLAN is like putting up walls to create separate rooms, except the walls are virtual, not physical. The same office, the same wiring, but now the marketing team is in one room, finance in another, and they can't go into each other's space.

Why bother? Two big reasons: tidiness and security. It keeps each team's traffic in its own lane so the network stays organized, and it means finance's data isn't floating around where the whole office can see it. The clever part is that all of this happens on the *same physical equipment*. No extra cables, no extra switches. You just draw the boundaries in software.

### Router-on-a-stick (Inter-VLAN routing)

VLANs are great at keeping rooms separate, but at some point two rooms *do* need to talk. Marketing has to send a file to finance. The whole point of a VLAN is that they can't reach each other directly, so something has to sit in the middle and pass traffic between them. That something is a router.

The obvious way would be to run a separate cable from the router to each VLAN, one port per room. That works, but it doesn't scale. Twenty VLANs would mean twenty cables and twenty ports. Router-on-a-stick is the clever shortcut: you use a *single* physical link between the switch and the router (the "stick"), carry every VLAN across it on a trunk, and then split that one interface into virtual sub-interfaces, one for each VLAN.

Think of it like a single hallway into a building where the router stands at the door wearing a different hat for each room. To traffic from VLAN 10 it acts as the VLAN 10 gateway; to VLAN 20 it acts as the VLAN 20 gateway. Same physical door, many roles. In Cisco IOS each role is just a sub-interface tagged with its VLAN:

```text
interface gig0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
interface gig0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
```

The `.10` and `.20` are the sub-interfaces, and `encapsulation dot1Q` is what lets the router read the VLAN tag on each frame so it knows which room a packet came from. Each sub-interface holds the gateway IP for its VLAN, so a device in VLAN 10 sends "off-network" traffic to `192.168.10.1`, the router catches it, and forwards it down to VLAN 20. One cable, one interface, every room connected.

![Photo of trunking](https://raw.githubusercontent.com/vathapann/lab/main/blog-posts/how-router-really-think/assets/IMG_5178.jpg)

### OSPF (Open Shortest Path First)

OSPF is one of the rules routers use to decide how your data travels across a network. Its job is to find the *best* route from one device to another, and it does this by building a complete map of the whole network. Every device and every connection and then calculating the smartest path across that map.

Let me put it in everyday terms. A network is just a bunch of devices connected together (phones, laptops, servers) and linked by WiFi, cables, or mobile cellular. When you send a message, it doesn't jump straight to its destination. It passes through a series of routers along the way and each router decides where to send the data to the next one.

Think of OSPF like the GPS in your car. Every router holds a full map and works out the fastest way to get your data from point A to point B. And here's the part that surprised me: "shortest" doesn't actually mean the fewest stops. It means the *fastest overall route*.

Just like Google Maps will recommend you a longer highway instead of a shorter road clogged with ten sets of traffic lights, OSPF will pick a route with more hops over fast links rather than fewer hops over slow ones. The goal isn't the fewest turns but arriving in the least time.

### NAT (Network Address Translation)

This one comes into play when we want to talk to the outside world. What I mean 'Outside World', I just mean the Internet lol.

Here's the problem NAT solves. Every device on your home network (your phone, laptop, TV) has its own private address, but those addresses only work **inside** your home. They mean nothing on the outside world. Think of them like apartment numbers in a building: "Apartment 4B" is useful inside the building, but a letter from another city needs the building's full street address to arrive.

NAT is the receptionist at the front desk. When your laptop sends a request out to the internet, NAT swaps your private address for the one public address your whole network shares. When the reply comes back, it remembers who asked and passes it to the right device. That's how a dozen gadgets in your house can all browse the internet at the same time while sharing a single public address.

Noted that NAT is available in IPv4 only. For IPv6, NAT is not necessary as IPv6 can cover a large number of public ip addresses.

## Challenges

About five weeks into the course, things finally started to click. I began to actually understand how a network works, how devices find each other and pass messages back and forth, and how switches quietly do a lot of that work behind the scenes.

The theory made sense on paper. Making it actually *work* in the lab was a different story and the trunk links were where I got stuck most of the time.

A "trunk" is the single cable that carries traffic for *all* the VLANs between two switches like one hallway connecting every room in the office. The catch is that both ends have to agree on exactly which rooms that hallway is allowed to serve. Two devices sitting on the same VLAN simply refused to ping each other across the two switches. I checked the IPs, the VLANs, the cables. Everything looked right.

It turned out the trunk wasn't allowing that VLAN through. The hallway existed, but VLAN 30 just wasn't on its list:

```text
interface gig0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

A quick `show interfaces trunk` confirmed it: only VLANs 10 and 20 were getting through. The fix was a single line:

```text
interface gig0/1
 switchport trunk allowed vlan add 30
```

One missing line of config, and an hour of staring at a screen convinced it was something far more complicated. That's the kind of lesson you only really learn by breaking it yourself.

![Photo of my concepts of networking put together into a project](https://raw.githubusercontent.com/vathapann/lab/main/blog-posts/how-router-really-think/assets/IMG_5991.jpg)


For weeks I'd been learning VLANs, routers, OSPF, and NAT as if they were unrelated topics on a syllabus. Then, building a full network from scratch, it clicked: these aren't four things, they're one system. The VLANs create the rooms, the router connects them, OSPF figures out the best way across, and NAT lets the whole thing reach the internet. Watching a packet make that entire journey, and *understanding every hop*, was genuinely one of the most satisfying moments of my degree so far.

Every concept from this course shows up in the cloud, just with different names. Designing an AWS VPC? That's subnets and routing. Setting up security groups? That's deciding who's allowed to talk to whom, the same idea as VLAN isolation. Debugging why one container can't reach another in Kubernetes? That's network policy, and it makes a lot more sense once you understand the fundamentals underneath it.

The difference is between "I followed the tutorial and it worked" and "I understand *why* it works, and I know where to look when it doesn't." That second one is what I want to bring to a team.

## Closing

Huge thanks to the teaching staff at UTS who made a genuinely complex topic approachable. Going back to the basics didn't slow me down. It gave me a foundation I'll build on for years. Next on the list: pushing deeper into cloud networking and Kubernetes, and finally tackling my CKA.

If you've made it this far, thank you for reading. And if you work in networking, DevOps, or platform engineering, I'd genuinely love to hear your feedback.
