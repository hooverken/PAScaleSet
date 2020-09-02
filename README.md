# Deploying Palo Alto NVA's into Azure for resilience and scalability

This ARM template (and its associated parameter file) are a very simple example of how to create a HA solution for using Palo Alto firewall NVA's in Azure.  

This general model should work for other NVA-type appliances that would benefit from being in a HA configuration and autoscaling.

## Why?

When we were standing up the Azure environment for my employer, we had a requirement that we use Palo Alto firewalls for security on the private network within Azure.  While PA has an appliance (NVA) version of their firewalls, the code they were running did not have any concept of failover or high availability, such as a way to have multiple NVA's work as a "team" and do things like distribute traffic among them or do automatic failover if one of the has to go down for some reason such as maintenance or a software fault.

### In our case, I realized a few things:

1. Firewalls that are placed behind a sufficiently capable load balancer don't have to be "aware" of each other *as long as their configurations match exactly*.
2. If we used a VM scale set to manage the appliances, we could create a "firewall cluster" that would scale out automatically when under stress and therefore be able to handle significant levels of load.  This would also save cost by not requiring us to run the NVA's on very large VM sizes as a hedge aginst worst-case traffic scenarios.
3. If we could solve the problem of how to get a NVA which is deployed automatically by the scale set to configure itself and join the load balancer pool unattended, we would have a setup which required no human intervention to scale up (or down) according to load -- and therefore we would have both a HA-like solution and not have to size the VM's used by the NVA's for a worst-case scanario to keep costs reasonable.

*Disclaimer: I am not a Palo Alto admin.  I relied on a member of our team who was one to handle the PA-specific elements of the environment when we built it.  Please talk to someone who knows PA's better than I do for questions about how some of the PA configuration elements described below were set up.*

## Ingredients of the Solution

The three Azure elements that are used in this example are:

* [Palo Alto virtual firewall appliances](https://www.paloaltonetworks.com/resources/datasheets/vm-series-for-microsoft-azure.html) (NVA's) from the Azure Marketplace
* An [Azure standard load balancer](https://docs.microsoft.com/en-us/azure/load-balancer/skus)
* An [Azure VM Scale set](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview)

## The Palo Alto NVA's

These are the same NVA's that can be [obtained through the Azure Marketplace](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/paloaltonetworks.vmseries-ngfw?tab=Overview).  Using the Pay-as-you-go license for the NVA's is recommended since using BYOL complicates deployment quite a bit.

You must make sure that the image(s) you are using are enabled for programmatic deployment, either by clicking on the associated link in the portal or following the instructions on [this blog post](http://blog.turtlesystems.co.uk/2018/10/16/Enabling-Programmatic-Access-in-Azure).

Since there will be a minimum of two NVA's deployed, it's (obviously) critical that the two firewalls have identical configurations to prevent weird behaviors.
If you plan to implement this solution, it's strongly recommended that you deploy a [Panorama](https://www.paloaltonetworks.com/network-security/panorama) server to manage the firewall NVA's as a unit since Panorama will make sure that all of the configurations match between the NVA's.

The configuration of the firewalls should align with the [health probe](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-custom-probe-overview) used by the load balancer so that the load balancer doesn't send traffic to a nonresponsive/not-yet-configured NVA.

It's also **very important** that the environment be set up so that when a new NVA is deployed it automatically associates itself with [Panorama](https://www.paloaltonetworks.com/network-security/panorama), otherwise the new NVA's will never be added to the pool by the load balancer.  **This is what makes the load balancer + scale set approach "work" -- the ability to have the scale set deploy a new NVA and for that running-but-unconfigured new node automatically configure itself and join the load balancer pool completely unattended.**  There are a few ways to bootstrap a freshly-deployed Palo Alto NVA but [this one](https://docs.paloaltonetworks.com/vm-series/8-1/vm-series-deployment/bootstrap-the-vm-series-firewall/bootstrap-the-vm-series-firewall-in-azure.html) which uses Azure Files seems to make the most sense.

As a bonus, you can save even more on the "run" cost with these NVA's by purchasing [reserved instances](https://azure.microsoft.com/en-us/pricing/reserved-vm-instances/) of course.

## The Azure Standard Load Balancer

It is the job of the load balancer in this environment to distribute traffic across the firewall NVA's that are part of the pool.  An [Azure Standard load balancer](https://docs.microsoft.com/en-us/azure/load-balancer/skus) is necessary since it can handle higher levels of traffic and has other advantages.

Per recommendation from an Azure networking "black belt" that I worked with while planning this sort of setup, the "front end" IP for the load balancer should be on a separate subnet from and of the interfaces used by the NVA's.

IMPORTANT: You must configure a [load balancer health probe](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-custom-probe-overview) that will query the correct interface on the firewall NVA in order for the LB to corrrectly "understand" "know" whether a firewall is ready or not.  This probe needs to be performed against the _interface which is handling traffic_, NOT the interface that is used for managing the NVA to prevent "fale-positive" results from the health probe.  In our case, we configured the PA to allow inbound connections to port 22 on the traffic-facing NIC but didn't attach any services to the port.  This allows a health probe to check for a successful connection on port TCP/22 and accept that as a signal that the NVA is ready to go.

Also, make sure that you have a NSG on the subnet used by the target interface which allows traffic from the [special IP used by the health probe service](https://docs.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16).

## The Azure VM Scale Set

In our design, we wanted to make sure that if there was a surge in traffic that risked overwhelming the bandwidth available to the firewalls, the solution would be able to deploy more capacity without human intervention and this is where the scale set comes in.

The configuration of the VM scale set is the same as for a non-NVA solution, EXCEPT that the image that is deployed by the scale set is the Palo Alto-provided NVA images.

In this case, the scale set rule is that if the aggregate CPU% of the set goes over 70% then a new node should be deployed.

When the new node is deployed, it bootstraps itself to connect to Panorama, which then sends configuration information to the new node.  That configuration package includes the port-22 setting described above so once the new node has accepted and applied the configuration it will go "green" on the health probe at which point it will start getting traffic directed to it from the load balancer.

