- choose a home region that is less crowded and has multiple avalability domains so you can access free resources easily (although right now resources are unavilable everywhere)
- you can only have Always Free resources ONLY IN YOUR HOME REGION (so choose less crowded home regions)

**Step 1: Create VCN - Crucial to create and launch instance**

Networking > Overview > Create a VCN with internet connectivity

**Step 2: Create Always Free instance**

Compute > Instances > Create Instance

**Placement:** Try every AD until you get resource allocated

In Advanced options, you can see Cluster Placement. Ensure it is in Fault Domain

**Image and Shape:**

**Change Shape:**

Choose Ampere

Choose VM.Standard.A1.Flex. Comes with an Always Free tag.

Free limits: 4 OCPUs and 24GB (you can use this in just 1 instance or use it across many instances, but this is the limit)

For n8n you need 3-4 OCPUs and 18-24GB for good performance

**Change Image:**

Ubuntu > Canonical Ubuntu 22.04

**Networking:**

Primary network > Select existing virtual cloud network

Subnet > Select existing subnet

Ensure you have a public IP address.

***Very Important: Remember to download your Private and Public keys for SSH.***

**Storage:**

Choose: Specify a custom boot volume size and performance setting

Free limits: upto 200GB across all instances. n8n works well at 70GB itself.

Review all configs once again and create instance.

It is very likely you get resource not available error if your domain is too crowded as Always Free resources are always on high demand.

Upgrading to Pay as you go lets you access Always Free Resources easily but requires credit card.
