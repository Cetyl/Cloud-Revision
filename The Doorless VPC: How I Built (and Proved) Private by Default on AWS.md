# The Doorless VPC: How I Built (and Proved) Private by Default on AWS

One VPC. Three tiers. No public entry point anywhere, not one. AWS Client VPN is the only way to reach the application or the database, and the security groups are written so that a successful connection is itself proof the traffic went through the tunnel, not just a coincidence that happened while the tunnel was up.

> Status: deployed. Built entirely through the AWS Management Console, region `ap-south-1`. Real Cost Explorer numbers and proof screenshots go into the marked placeholders below once pulled together.

## Why this exists

Almost every VPN walkthrough on the internet ends at the same place: a green checkmark that says Connected. That checkmark is a real achievement, it means a tunnel came up, a handshake completed, keys were exchanged. But it answers a much narrower question than most people think it does. It tells you the tunnel exists. It does not tell you that any particular request, your app call, your database query, your SSH session, actually travelled through it instead of some other path.

Those are two different claims, and conflating them is where a lot of "secure" architectures quietly stop being secure. A developer connects the VPN, opens their laptop, everything works, they assume the VPN is doing the job. Nobody goes back and checks whether the traffic that just worked would have worked anyway, VPN or no VPN, because a security group somewhere still allows `0.0.0.0/0`.

I wanted to build something where that gap couldn't exist by construction, not just by discipline. So the design constraint that drives this whole project is simple to state and, it turns out, surprisingly rare to actually enforce: **every security group in the VPC allows exactly one source, the VPN client CIDR, and nothing else, anywhere.** Not a broader range "just in case." Not a rule left over from testing. One source, everywhere, on purpose.

Once that constraint is real, something useful falls out of it for free: a successful connection stops being circumstantial evidence and becomes a proof. If the app loads, the only thing that could have produced a permitted source address is the VPN. There is no other path through which that packet could have been allowed in. That is the idea this whole build exists to demonstrate, end to end, with evidence at every layer, not just at the client.

## What "private by default" actually means here

It is tempting to treat "private subnet" as a label you attach to a subnet in the console. It isn't. A subnet is private only because of what its route table points to, and this project makes that literal by giving each of its three tiers a genuinely different route table, not a copy of the same one with different names.

**The public tier** holds exactly two things: the internet gateway and the NAT gateway (plus, in the bonus phase, a Pritunl box for comparison). It runs no application code. This matters more than it sounds like it should, because the most common misreading of "public subnet" is that it's where the app goes. It isn't. It's where the doors go. Nothing about being in a public subnet requires exposing an application to it, and this build proves that by keeping the app entirely somewhere else.

**The private app tier** can talk outward, through the NAT gateway, for OS patching and package installs, but nothing can talk inward except through the Client VPN endpoint's ENI. This is the tier where the interesting security group work lives: the internal ALB, the app instances, and the VPN's own network interface all sit here, and every one of them is scoped to the client CIDR.

**The private data tier** is the one most tutorials skip, because it's the least glamorous and the most important. It has no NAT route and no internet gateway route at all. Not "restricted," not "firewalled off," genuinely absent from its route table. That means the database cannot dial out even if something on it were compromised and tried to. The honest cost of this is that the instance can't reach public package repositories either, which is a real operational tradeoff worth naming rather than hiding: in production, that gap gets closed with VPC endpoints for the specific AWS APIs the tier actually needs, not with a NAT route "just to be safe." RDS sidesteps a chunk of this problem because AWS patches the engine itself, which is one of the more underrated arguments for a managed data service over running your own database on EC2.

Three tiers. Three route tables. Public versus private is a consequence of what's written in a route table, not a checkbox ticked somewhere in a wizard. See `private-by-default-architecture.svg` for the full diagram.


<img width="1624" height="1085" alt="AWS Client VPN VPC Architecture Diagram" src="https://github.com/user-attachments/assets/2bc4a967-96db-4f67-ac8d-05a6c5a004c6" />


## Why AWS Client VPN, specifically

I looked at three other options before settling on this one, and it's worth saying why, because the choice shapes what the project can teach.

A self-hosted WireGuard box on a `t4g.nano` is dramatically cheaper, a few cents an hour instead of ten, and it's genuinely the better answer if the goal were minimizing cost. But it hides nothing, there's no separate authorization layer to demonstrate, no distinction between "routable" and "permitted" to break on purpose, because WireGuard collapses both into one `AllowedIPs` line. That makes it a worse teaching vehicle for exactly the thing this project is about.

Pritunl gets a fair shake here too, as the bonus comparison in the build, because it has a real web console and real user lifecycle management, things AWS Client VPN's mutual-TLS mode genuinely lacks. But its site-to-site and multi-server features sit behind a paid tier, and out of the box it's still wrapping OpenVPN or WireGuard underneath, so it doesn't replace the core concept, it decorates it.

AWS Client VPN earns its place because it's the one option where "the endpoint has its own route table" and "authorization is a separate decision from routing" are first-class, visible, breakable concepts in the console, not implementation details buried in a daemon's source code. That separation is the single most valuable thing this project has to teach, and it only exists because Client VPN was built to be operated by teams who need per-group access policies, not just a working tunnel.

## Two decisions that are easy to get backwards

**Mutual TLS instead of SAML.** SAML through IAM Identity Center is closer to how a real company would run this, with a self-service portal and group-based policy. I chose mutual TLS anyway, because building a certificate authority by hand with `easy-rsa`, understanding why a server cert and a client cert signed by the same CA collapse into a single ACM import, and watching a revoked certificate keep working because nobody uploaded a CRL, is a far richer lesson than clicking through an identity provider setup wizard. The tradeoff is honest: this build does not scale to a team, because every new user is a manual certificate issuance. That's named directly in Known Limitations below, not glossed over.

**Split tunnel on, against the AWS default of off.** With split tunnel enabled, only traffic bound for `10.0.0.0/16` enters the tunnel; everything else, your regular browsing, leaves your laptop exactly as it would without the VPN running at all. With it off, every packet you send, video calls included, gets forced through the NAT gateway in this lab and billed at NAT data processing rates. I run with it on for cost and for correctness of the demo, then flip it off deliberately as one of the break-it-on-purpose exercises, specifically so the difference is something you watch happen rather than something you take on faith.

## The proofs, and why each one is there for a different reason

Six is more than strictly necessary. That's on purpose. Each proof answers a different skeptic.

**1. The security group design itself.** This is the one a security reviewer actually wants, because it's structural rather than observational. It doesn't say "the traffic looked like it came through the VPN this one time," it says "there is no configuration under which this traffic could have arrived any other way." That's a stronger claim than a log entry, because a log entry only tells you what happened, not what could happen.

**2. VPC flow logs.** This is the one an auditor wants, because it's a durable, queryable record rather than something you watched happen live. Filtering the app tier's ENI and reading `srcaddr` shows `172.16.0.x` on every accepted connection, and shows nothing from a public range, ever, across the log's entire retention window.

**3. A packet capture on the application instance**, taken through EC2 Instance Connect's browser terminal rather than SSH from the open internet, which would itself be a violation of the design. This is the one that convinces someone who trusts the server over the network: the packets, as the app instance itself receives them, carry the client CIDR as their source.

**4. A packet capture on the laptop, on two interfaces at the same time.** This is the one that makes the abstraction physical. On `en0`, the real network interface, you see UDP 443 to the endpoint's public address and nothing else, the payload fully opaque. On `utun`, the interface the VPN client creates, you see the same request in cleartext HTTP. Same packet, captured twice, once encrypted and once not, and the difference between them is the entire value proposition of a VPN made visible in two terminal windows side by side.

**5. The database's own opinion.** Running `select client_addr, application_name from pg_stat_activity;` over the tunnel, in a GUI SQL client, has the database itself report the connecting address as `172.16.0.x`. PostgreSQL has no stake in whether a VPN is being used, so its own bookkeeping saying so carries a kind of neutral weight the other proofs don't.

**6. The laptop's own routing table, before and after.** `netstat -rn | grep 10.0` returns nothing before connecting, and shows a `10.0.0.0/16` route through a `utun` interface after. This is the proof that needs no AWS access at all, only your own machine, which makes it the easiest one for a reader to actually replicate rather than take on faith.


## The three things that must all agree, and how each failure looks different

For traffic to reach a private resource, three independent conditions have to hold at once, and knowing which one failed, from the outside, with only the symptom to go on, is the actual skill this project is trying to build.

1. **The Client VPN endpoint's own route table** must have a route to the destination. This table is separate from every VPC route table, which surprises people the first time they go looking for it.
2. **An authorization rule** must permit the client to reach that destination CIDR. A route existing does not imply permission exists; they are set in entirely different places in the console.
3. **The security group and the network ACL** must both permit it, and they don't behave the same way: the security group is stateful, so a permitted inbound request gets its response allowed automatically, while the NACL is stateless and evaluates both directions independently, including the ephemeral return port.

| Broken on purpose | What the client sees | Where to look |
|---|---|---|
| Authorization rule deleted | Connection stays healthy and green, traffic to that CIDR simply times out | Endpoint authorization rules |
| Endpoint route deleted | Traffic never enters the tunnel at all, there's no local route for it | Local routing table, endpoint route table |
| Security group blocking | Timeout or connection refused, depending on the protocol | VPC flow logs show REJECT on the ENI |
| NACL blocking | Timeout, but flow logs show REJECT in only one direction | NACL rule numbers, and remember ephemeral ports on the return leg |
| Client cert revoked, no CRL uploaded to the endpoint | Connects completely normally, which is exactly the danger | Endpoint's client certificate revocation list setting |

That last row deserves its own sentence, because it's the one people are least likely to already know: **revoking a certificate and cutting off access are two separate actions.** A revoked cert only stops working once its serial number reaches the endpoint's CRL. Until then, from the endpoint's point of view, nothing has changed.

## How it was built, step by step

Everything below was done by clicking through the AWS Management Console, not the CLI. Two moments are genuine exceptions, because AWS offers no console tool for them at all, and both are called out where they happen. The full click-by-click version, screen by screen, field by field, lives in `private-by-default-console-guide.md`; this is the substance of it, condensed.

**Phase 0, guardrails.** A Cost Explorer budget alarm set to $10/month with an 80% threshold, created before any billable resource exists, because the one line item in this build that quietly becomes real money is the Client VPN subnet association, which bills whether or not anyone is connected.

**Phase 1, the network.** One VPC, `10.0.0.0/16`. Six subnets across two AZs, two per tier. Three route tables, one per tier, each genuinely different: public routes `0.0.0.0/0` to the internet gateway, app routes `0.0.0.0/0` to the NAT gateway, data has no default route at all. VPC Flow Logs enabled on the app tier subnets from the start, so later phases have real evidence rather than a promise of evidence.

**Phase 2, the workload, proven unreachable.** An internal Application Load Balancer with no public IP, two app instances with no public IP and no key pair (there is nothing to SSH into from the internet, on purpose), and an RDS PostgreSQL instance with public accessibility explicitly disabled. Three security groups built so that not one rule anywhere references `0.0.0.0/0`. The phase ends by opening the ALB's DNS name in a browser with no VPN connected and watching it fail, on purpose, as the baseline everything else gets measured against.

**Phase 3, the certificate authority.** The one unavoidable terminal step in the entire build, run in AWS CloudShell so it stays inside the browser rather than touching a local machine. A self-managed CA via `easy-rsa`, a server certificate, a client certificate, both signed by the same authority. Only the server certificate needs importing into ACM, because the shared CA means one ACM entry validates both directions of the handshake.

**Phase 4, the endpoint, and the first real connection.** The Client VPN endpoint, client CIDR `172.16.0.0/22`, mutual TLS authentication pointed at the ACM certificate from Phase 3. One subnet association, in the app tier, which is the moment the hourly charge starts. One authorization rule permitting the VPC CIDR. The exported client configuration, hand-stitched with the client certificate and key. Then the proof that closes the loop: the same browser tab that failed in Phase 2 now loads the app.

**Phase 5, the proofs.** All six, captured as evidence: the security group screens, a flow log query, a packet capture on the app instance through EC2 Instance Connect, a packet capture on the laptop across two interfaces at once using Wireshark, a `pg_stat_activity` query through a GUI SQL client, and the laptop's own routing table before and after connecting.

**Phase 6, breaking it on purpose.** Five failures, staged and reverted one at a time: split tunnel disabled and re-enabled while watching a public IP change, the authorization rule deleted and restored, the endpoint route deleted and restored, a NACL rule added to block the app tier and removed again, and a client certificate revoked without a CRL, reconnected successfully to prove the gap, then fixed by uploading the CRL and reconnecting to watch it get refused.

**Phase 7, the Pritunl comparison.** The same private resources, reached instead through Pritunl running on its own EC2 instance, to compare setup time, whether revoking a user in Pritunl's web console actually cuts access immediately, against Client VPN's manual certificate workflow.

**Phase 8, teardown.** Reverse order, because an attached NAT gateway or load balancer blocks VPC deletion: disassociate the Client VPN subnet first, since that's what actually stops the hourly charge, then delete the endpoint, the load balancer, the instances, the RDS database, the NAT gateway, release its Elastic IP, detach and delete the internet gateway, delete the subnets and route tables, then the VPC itself.

## Services used

AWS Client VPN, Amazon VPC, Amazon EC2, Amazon RDS for PostgreSQL, Application Load Balancer (internal), AWS Certificate Manager, VPC Flow Logs, Amazon CloudWatch Logs, AWS CloudShell (certificate generation only), EC2 Instance Connect. Bonus phase: Pritunl on EC2, for a side by side comparison.

## Cost

Design-time estimates, `ap-south-1` list pricing checked August 2026.

| Item | Rate | Notes |
|---|---|---|
| Client VPN subnet association | ~$0.10/hr | **Billed whether or not anyone is connected.** |
| Client VPN connection | ~$0.05/hr | Only while a client is actually connected |
| NAT gateway | ~$0.056/hr | Plus per GB processed |
| Application Load Balancer | ~$0.025/hr | Plus LCU charges, negligible here |
| RDS db.t4g.micro | ~$0.017/hr | Single AZ |
| EC2 instances | ~$0.025/hr total | Two t3.micro, plus Pritunl in the bonus phase |

Estimated at roughly $2.50 to $3.50 for a full build and demo day. Left running for a month, roughly $170, of which about $72 is the Client VPN endpoint alone, sitting idle with nobody connected.

> `[ real Cost Explorer figure goes here once pulled ]`

**Disconnecting the VPN client does not stop the endpoint charge.** The billed object is the subnet association. Teardown means disassociating the subnet and deleting the endpoint, not closing the VPN client.

## Known limitations

- **Single VPC only.** No VPC-to-VPC connectivity, site to site IPsec, or peering in this build. Deliberately cut from scope to keep the proof story tight rather than diluting it across a second network.
- **No high availability on the VPN.** One subnet association means one AZ. A production build would associate a subnet per AZ and pay for both.
- **No SAML or directory integration.** Mutual TLS means certificate issuance and revocation are manual, which does not scale past a handful of users. A real limitation, not a lab shortcut, and part of why the Pritunl comparison exists at all.
- **The certificate revocation gap is demonstrated, not fixed.** Revoking a client cert without uploading a CRL to the endpoint does not stop that client connecting. Shown deliberately, because it is a genuinely dangerous thing to not know about a system you're operating.
- **No infrastructure as code.** Built by hand through the console so each object is created and inspected deliberately, at the cost of not being trivially reproducible.
- **The data tier cannot reach the internet in either direction**, so it cannot patch itself from public repositories. RDS sidesteps this because AWS patches the engine directly.

## Licence

MIT
