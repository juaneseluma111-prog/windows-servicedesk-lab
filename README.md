# Windows Domain & Service Desk Lab

I built this to get hands-on with the things an IT service desk
actually does every day — creating users, resetting passwords,
unlocking accounts, granting access through groups, and pushing
policy out to machines. Reading about it wasn't enough.

## Setup

| Machine | Role | IP | OS |
|---|---|---|---|
| DC01 | Domain controller and DNS | 192.168.56.10 | Windows Server 2019 |
| WindowsIT | Client | 192.168.56.20 | Windows 10 Pro |

Both run in VirtualBox with two adapters each: NAT so they have
internet, and an internal network called `labnet` that carries the
domain traffic. The internal adapters have static IPs; the NAT ones
stay on DHCP.

![Server network config](screenshots/01.1-vbox-network-config.png)
![Client network config](screenshots/01.2-vnox-network-client.png)
![Server ipconfig](screenshots/02-server-ipconfig.png)
![Client ipconfig](screenshots/03-client-ipconfig.png)

## What I set up

I promoted the server to a domain controller for a new forest,
`lab.local`, which also makes it the DNS server for the domain.

Then I built out the structure: OUs for Finance, IT and Sales, users
inside each one, and security groups. Access goes through group
membership, not by giving permissions to individual accounts — that's
how it works anywhere real, and it's the difference between a five
second job and a nightmare when someone changes department.

![AD structure](screenshots/05-ad-structure.png)
![Users and groups](screenshots/06-users-groups.png)

I created a Group Policy Object on the Finance OU that blocks users
from changing their desktop background. Not because that matters, but
because it proves the mechanism works — one setting, applied to a
whole department, without touching a single machine.

![Group Policy](screenshots/07-Group-Policy-Finance.png)

Then I joined the Windows 10 client to the domain and logged in as a
domain user.

![Domain join](screenshots/08-domain-join.png)
![Domain login](screenshots/09-domain-login-escritorio-amurphy.png)
![whoami](screenshots/10-whoami.png)

The policy came through, which I checked both by trying to change the
background and by running `gpresult`.

![GPO applied](screenshots/11-gpo-applied.png)
![gpresult](screenshots/12-gpresult.png)

And the machine shows up in Active Directory under Computers.

![Computer in AD](screenshots/13-computer-in-ad.png)

## The service desk tasks

These are the four that come up constantly:

| What I did | The ticket it answers |
|---|---|
| Reset a password | "I can't remember my password" |
| Unlock an account | "I typed it wrong too many times" |
| Added a user to a security group | "I need access to the Finance folder" |
| Disabled an account | Someone left the company |

Worth saying on that last one: you **disable** a leaver's account, you
don't delete it. The data and the mailbox have to stay. Deleting it is
the kind of mistake you only make once.

![Password reset](screenshots/15-password-reset.png)
![Unlock account](screenshots/16.2-unlock-account.png)
![Disable account](screenshots/16.1-dissable-account.png)
![Account disabled](screenshots/16-account-dissabled.png)

## What went wrong

Five things broke along the way. I learned more from these than from
the parts that worked first time, so I'm documenting all of them.

**The static IP ended up on the wrong adapter.** When I added the
second adapter in VirtualBox, Windows treated it as new hardware and
my static config stayed on the NAT interface — which also killed
internet on both machines. What gave it away was the internal adapter
sitting on a 169.254.x.x address. That's Windows saying "I asked for
an IP and nobody answered", which is exactly what happens on an
isolated network with no DHCP.

![IPv4 parameters](screenshots/17-ip4-parameters.png)
![Client internal adapter](screenshots/18-windows-cliente-eth2.png)

**DNS was going to my home router instead of the domain controller.**
With two adapters, Windows picked the NAT one for DNS, so
`nslookup lab.local` came back as a non-existent domain — my router
obviously has no idea what `lab.local` is. Fixed it by setting the
connection-specific DNS suffix on the internal adapter and changing
the interface metrics, 10 on the internal one and 50 on NAT, so the
domain controller gets asked first.

![Client DNS options](screenshots/18-advanced-options-client.png)

**The client was Windows 10 Home, which can't join a domain at all.**
The Domain option is just greyed out in System Properties. I tried an
in-place upgrade to Pro and it failed with 0x803fa067 before it
eventually went through. This one is worth knowing: if someone brings
their own laptop into a company, there's a good chance it's a consumer
edition and it can't be domain-joined until it's upgraded or
reimaged. The upgrade also wiped my network settings, so I had to
redo them.

![Upgraded to Pro](screenshots/19-version-changed-w10pro.png)

**The firewall was blocking domain traffic.** Windows Server doesn't
answer ping by default, and joining a domain needs a lot more than
ping anyway — LDAP, Kerberos, SMB. I turned the firewall off to rule
it out and turned it straight back on once the client had joined.
Turning off a firewall is how you find a problem, not how you fix one.

![Firewall restored](screenshots/14-firewall-restored.png)

**The domain controller was advertising addresses the client couldn't
reach.** This was the one that took longest and taught me the most.
The DC had registered all of its addresses in DNS — including the NAT
address, 10.0.2.15, and IPv6. The client could resolve the name
perfectly but still couldn't join, and the error said it: the SRV
record correctly pointed at `dc01.lab.local`, but no domain controller
could be contacted. It was resolving to addresses that don't exist on
the internal network. I disabled IPv6 on the lab adapters and
unchecked *Register this connection's addresses in DNS* on the NAT
interface so the DC only advertises the address that actually works.

![DNS registration disabled](screenshots/17-dns-register-unchecked.png)
![Name resolution](screenshots/20-nslookup.png)

## What I'd take into a real job

Nearly everything that broke here was either DNS or a setting applied
to the wrong network adapter. If something in a domain isn't working,
those are the first two places I'd look now.

The single most useful thing I picked up was the difference between
two ping errors. "Destination host unreachable" means there's no route
to the target. "Request timed out" means the packets are going out and
nothing is answering — so it's a firewall or a service that isn't
running, not a routing problem. When my ping changed from one message
to the other, that told me the fault had moved from the client to the
server, and I stopped looking in the wrong place.

The other lesson was from the last problem: a name resolving doesn't
mean the address behind it is reachable. I'd been treating a
successful `nslookup` as proof the network was fine, and it wasn't.
Reading the full error detail instead of the summary line was what
finally showed me why.