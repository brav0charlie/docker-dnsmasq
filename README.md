# brav0charlie/docker-dnsmasq
Docker container with a simple installation of dnsmasq to provide DNS and DHCP services to your local network.

Based on Alpine Linux 'Edge' and dnsmasq-2.84-r0.

:bangbang: This base OS this container uses is Pre-Release Software. Please don't run this in production and expect it to perform. I built this for my home network, and I live alone. I won't be harmed if my network goes down. :)

## About
This project was born out of a larger project to get rid of the large, loud, hot server sitting next to my desk at home. At the time, I was running a Windows Active Directory environment inside VMware, and needed to provide DNS/DHCP to my local network after before getting rid of the server. I don't like hosting it on my router for no good reason at all, and I'm in the progress of learning Docker so I figured I'd take a stab at a lightweight DNS/DHCP container. This container has been providing all DNS and DHCP services to my home network since late July 2020.

I'm not really doing anything groundbreaking here, there really isn't much to the Dockerfile at all. Most of my work has been in building the custom configuration files to make things easier for me to manage later with any text editor I'd like.

## Important Notes
- Currently, this container only listens on IPv4.
- We're using Quad9 for our upstream DNS provider by default, but this can be changed.
  - Details on changing this are in `config/dns.conf`.
- Container binds to DNS & DHCP ports of the host machine (i.e., 53/udp, 67/udp, 68/udp).
  - Ubuntu 18.04 and later use a stub resolver (essentially dnsmasq) to cache DNS requests from the local host, so you'll need to disable that before starting this container:
    - `sudo systemctl stop systemd-resolved && sudo systemctl disable systemd-resloved`

## Installation
1. Modify the four files in `config` to suit your environment. The files are commented well enough that you can figure out what's needed.
2. Run docker-compose up -d docker-compose.yml to build & start the container.

## Configuration
It's strongly recommended that you read the dnsmasq docs (http://www.thekelleys.org.uk/dnsmasq/doc.html) and visit the main site for more information on dnsmasq. While not entirely necessary, as the documentation below pretty well spells out the basics, it will give you a better understanding of how dnsmasq works it's magic. 

The four files provided in the `config` folder make up the core of our configuration. 

The first file is `dns.conf`, which contains our dnsmasq options related to DNS and the main application itself.

The second file is `dhcp.conf`, which contains our DHCP configuration. This is where we define our DHCP scopes, default gateways, and DNS servers.

The third file is `hosts.local`, and is referenced in addition to `/etc/hosts` (in the container). This is a standard hosts file, and is where we define static DNS entries. DHCP client hostnames are automatically added to the server, there's no need to define them here.

Finally, the fourth file is `dhcp-reservations.conf`. This is where we will define our DHCP reservations. 

*Syntax for each file is noted in the comments & examples in each file.*

:bangbang: After changing any of the config files, you'll need to restart the container to pick up the changes (`docker container restart dnsmasq`). The same goes for dnsmasq installed directly on a host: you have to do a `sudo systemctl restart dnsmasq` to pick up the changes.

### DNS Configuration
In `dns.conf` we set some options:
1. `no-resolv`
   1. This tells dnsmasq to ignore the /etc/resolv.conf file and only forward to the servers we define later in this file.
2. `addn-hosts=/etc/dnsmasq.d/hosts.local`
   1. This is the hosts file for our domain. Here, you'll define the static IP mappings, just like you would in /etc/hosts.
3. `server=9.9.9.9` & `server=142.112.112.112`
   1. These are the primary and secondary DNS servers provided by the service Quad9. Visit https://quad9.com for details. 
   1. You may also change these to whatever you'd like. For example, 8.8.8.8 & 8.8.4.4 for Google DNS, 1.1.1.1 or 1.0.0.1 for CloudFlare's 1.1.1.1 service.
4. `domain-needed`
   1. Tells dnsmasq not to forward plain names (i.e., hostnames without a FQDN) to upstream servers.
5. `bogus-priv`
   1. Tells dnsmasq not to forward private address space (i.e., 10.0.0.0/8, 192.168.0.0/16, 172.16.0.0/12) requests upstream. This prevents reverse-DNS lookups of internal ranges from being sent to external servers (i.e., the internet).
6. `local=/xxxx/`
   1. Tells dnsmasq that this domain is a private domain and should only be served from /etc/hosts and our hosts.local file.
   1. You can do as many of these as you'd like, one per line. This is helpful if you experiment with multiple internal domain names or subdomains.
7. `domain=internal.example.com`
   1. Set your primary internal domain name here. The example uses the subdomain internal.example.com.
8. `expand-hosts`
   1. Tells dnsmasq to expand queries for a just a hostname to a FQDN by appending the domain set above.
9. `log-facility=/var/log/dnsmasq/dnsmasq.log`
   1. Tells dnsmasq to write logs to a file. We specify the file, because we bind-mount the the directory so we can access the file on our physical host.
   2. Dnsmasq will close and reopen it's log files when it receives the signal SIGUSR2. This is helpful if you'd like to use something like `logrotate` to rotate the logs. Just add a post-rotation command of `docker kill --signal=SIGUSR2 dnsmasq` to your logrorate script on the host.
9. `log-queries`
   1. Tells dnsmasq to log all queries it handles. This can cause lots of writes to disk, so be careful when using this flag. Uncomment to enable.


### DHCP Configuration
In `dhcp.conf`, we'll set up our DHCP scopes:
1. Set up scopes
   1. In this section, you'll add each IP range, or scope, that you want dnsmasq to serve addresses from. 
   1. You can do as many as you'd like, one per line.
   1. The syntax is:
      1. `dhcp-range=<range_name>,<start_address>,<end_address>,<subnet_mask>,<leasetime>` 
      1. Leasetimes can be expressed in m, h, or d, or 'infinite' (i.e., 12h)
1. Set gateway servers
   1. In this section, you'll define the default gateway, or router, for each DHCP scope.
   1. Each scope should have one entry, one entry per line.
   1. The syntax is:
      1. `dhcp-option=<range_name>,3,<ip_address>` (the 3 tag indicates it's a router)
1. Set DNS servers
   1. Here, we set our DNS servers for each scope. It's recommended you set at least a primary server, likely the address of the server you'll be running this container on, in fact!
   1. The syntax is:
      1. `dhcp-option=<range_name>,6,<ip_address>` (the 6 tag indicates it's a DNS server)

### Static DNS Entries
We set our static DNS entries in the `hosts.local` file. This file works just like /etc/hosts. 

The syntax is:
`<ip_address>     <hostname>`

The file accepts standard comments using the hash (#) symbol. It's recommended you comment your files so you know what you're looking at later. For example:
```
# This is my desktop computer
192.168.1.100     example-host1.internal.example.com
# This is my laptop
192.168.1.101     example-host2.internal.example.com
```

### DHCP Reservations
Finally, we set our DHCP reservations in `dhcp-reservations.conf`:
- You can do as many as you'd like, one entry per line. The IP address you're reserving *does not* need to be within one of the scopes you defined above, just within the range of the target subnet.
- Also supports standard comments using the hash (#) symbol. Comment your files!
- The syntax is:
  - `dhcp-host=<mac_address>,<ip_address>,<hostname>,<leasetime>`
  - MAC Addresses should be entered in AB:CD:EF:G1:23:45 format (not case-sensitive)
  - Leasetimes can be expressed in m, h, or d, or 'infinite' (i.e., 12h)
    
    
## Tips
#### Source Version Control
I keep my config files under source version control linked to a private repo. This way, I can keep a record of all changes I've made and easily roll back to a previous version of my config files. 

This can be done pretty easily. First, create a new repo on github and call it whatever you'd like. Then, cd into your config directory, and run the following, replacing the `<url>` with either `https://github.com/<username>/<repoistory_name>` or `git@github.com:<username>/<repository>`:
```bash
$ git init
$ git add .
$ git commit -m 'message'
$ git remote add origin <url>
$ git push -u origin main
```

# License Information
I've included the GPLv3, as that is what dnsmasq uses.

We're not doing anything with the dnsmasq code, so I'm not sure it even applies here, but here we are.

As for the contents of this git repo, do whatever the hell you want with it. :)
