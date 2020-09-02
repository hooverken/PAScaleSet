# Yale Replica Network Environment for Routing/FW testing

This template deploys an isolated network environment with the following configuration:

## Virtual Networks (VNETs)

### Hub VNET

This VNET simulates the Shared Services VNET in production with IP space 10.11.0.0/22

#### Subnets

- **Hubsubnet1** IP space 10.11.0.0/24 (NonsecuresubnetRouteTable)
- **Hubsubnet2** IP space 10.11.1.0/24 (NonsecuresubnetRouteTable)
- **GatewaySubnet** IP space 10.11.3.0/24

#### Virtual Machines

- **HubVM1** (WS2016) on HubSubnet1

#### Virtual Network Gateways

- **HubNetworkVNG** on GatewaySubnet with connection to **ExternalVNET**


#### Connectivity

- Peering to Spoke VNET 1 (simulated downstream peered VNET 1)
- Peering to Spoke VNET 2 (simulated downstream peered VNET 2)
- VNG connection to ExternalVNET (simulated ExpressRoute relationship to Yale internal network)


### Spoke VNET 1

This VNET simulates a VNET which is peered with Shared Services VNET in production and uses IP space 10.11.4.0/22

#### Subnets

- **SpokeVNET1-Secure** IP space 10.11.4.0/24 (SecureSubnetRouteTable)
- **SpokeVNET1-Nonsecure** IP Space 10.11.5.0/24 (NonsecuresubnetRouteTable)

#### Virtual Machines

- **Spoke1SecureVM** (WS2016) on subnet **SpokeVNET1-Secure**
- **Spoke1NSVM** (WS2016) on subnet **SpokeVNET1-NonSecure**

#### Connectivity

- Peering to Hub VNET


### Spoke VNET 2

This VNET simulates a VNET which is peered with Shared Services VNET in production and uses IP space 10.11.8.0/22

#### Subnets

- **SpokeVNET2-Secure** IP space 10.11.8.0/24 (SecureSubnetRouteTable)
- **SpokeVNET2-Nonsecure** IP Space 10.11.9.0/24 (NonsecuresubnetRouteTable)

#### Virtual Machines

- **Spoke2SecureVM** (WS2016) on subnet **SpokeVNET2-Secure**
- **Spoke2NSVM** (WS2016) on subnet **SpokeVNET2-NonSecure**

#### Connectivity

- Peering to Hub VNET


### External VNET

This VNET simulates an external network which is connected by something other than peering. and uses IP space 192.168.0.0/22

#### Subnets

- **externalsubnet1** IP space 192.168.1.0/24 (NonsecuresubnetRouteTable)
- **GatewaySubnet** IP Space 10.11.2.0/24

#### Virtual Machines

- **ExternalVM** (WS2016) on subnet **ExternalSubnet1**

#### Virtual Network Gateways

- **ExternalNetworkVNG** on GatewaySubnet with connection to **HubVNET**
