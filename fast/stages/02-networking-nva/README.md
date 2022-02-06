# Networking with Network Virtual Appliance

This stage sets up the shared network infrastructure for the whole organization.

It is designed for those who would like to leverage Network Virtual Appliances (NVAs) between trusted and untrusted areas of the network, for example for Intrusion Prevention System (IPS) purposes.

It adopts the common “hub and spoke” reference design, which is well suited for multiple scenarios, and it offers several advantages versus other designs:

- the "trusted hub" VPC centralizes the external connectivity towards trusted network resources (e.g. on-prem, other cloud environments and the spokes), and it is ready to host cross-environment services like CI/CD, code repositories, and monitoring probes
- the "spoke" VPCs allow partitioning workloads (e.g. by environment like in this setup), while still retaining controlled access to central connectivity and services
- Shared VPCs -both in hub and spokes- split the management of the network resources into specific (host) projects, while still allowing them to be consumed from the workload (service) projects
- the design facilitates DNS centralization

Connectivity between the hub and the spokes is established via [VPC network peerings](https://cloud.google.com/vpc/docs/vpc-peering), which offer uncapped bandwidth, lower latencies, at no additional costs and with a very low management overhead. Different ways of implementing connectivity, and related some pros and cons, are discussed below.

The diagram shows the high-level design and it should be used as a reference throughout the following sections.

The final number of subnets, and their IP addressing will depend on the user-specific requirements. It can be easily changed via variables or external data files, without any need to edit the code.

<p align="center">
  <img src="diagram.svg" alt="Networking diagram">
</p>

## Design overview and choices

### Multi-regional deployment

The stage deploys the the infrastructure in two regions. By default, europe-west1 and europe-west4. Regional resources include NVAs (templates, MIGs, ILBs) and test VMs.
This provides enough redundancy to be resilient to regional failures.

### VPC design

The "landing zone" is divided into two VPC networks:

- the trusted VPC: the connectivity hub towards other trusted networks
- the untrusted VPC: the connectivity hub towards any other untrusted network

The VPCs are connected with two sets of sample NVA machines, grouped in regional (multi-zone) [Managed Instance Groups (MIGs)](https://cloud.google.com/compute/docs/instance-groups). The appliances are plain Linux machines, performing simple routing/natting, leveraging some standard Linux features, such as *ip route* or *iptables*. The appliances are suited for demo purposes only and they should be replaced with enterprise-grade solutions before moving to production.
The traffic destined to the VMs in each MIG is mediated through regional internal load balancers, both in the trusted and in the untrusted networks.

By default, the design assumes the following:

- on-premise networks (and related resources) are considered trusted. As such, the VPNs connecting with on-premises are terminated in GCP, in the trusted VPC
- the public Internet is considered untrusted. As such [Cloud NAT](https://cloud.google.com/nat/docs/overview) has been deployed in the untrusted landing VPC only
- cross-environment traffic and traffic from any untrusted network to any trusted network (and vice versa) pass through the NVAs. For demo purposes, the current NVA performs simple routing/natting only
- any traffic from a trusted network to an untrusted network (e.g. Internet) is natted by the NVAs. Users can configure further exclusions

The trusted landing VPC acts as a hub: it bridges internal resources with the outside world and it hosts the shared services consumed by the spoke VPCs, connected to the hub thorugh VPC network peerings. Spokes are used to partition the environments. By default:

- one spoke VPC hosts the development environment resources
- one spoke VPC hosts the production environment resources

Each virtual network is a [shared VPC](https://cloud.google.com/vpc/docs/shared-vpc): shared VPCs are managed in dedicated *host projects* and shared with other *service projects* that consume the network resources.
Shared VPC lets organization administrators delegate administrative responsibilities, such as creating and managing instances, to Service Project Admins while maintaining centralized control over network resources like subnets, routes, and firewalls.

Users can easily extend the design to host additional environments, or adopt different logical mappings for the spokes (for example, in order to create a new spoke for each company entity). Adding spokes is trivial and it does not increase the design complexity. The steps to add more spokes are provided in the following sections.

In multi-organization scenarios, where production and non-production resources use different Cloud Identity and GCP organizations, the hub/landing VPC is usually part of the production organization. It establishes connections with the production spokes within the same organization, and with non-production spokes in a different organization.

### External connectivity

External connectivity to on-prem is implemented leveraging [Cloud VPN HA](https://cloud.google.com/network-connectivity/docs/vpn/concepts/topologies) (two tunnels per region). This is what users normally deploy as a final solution, or to validate routing and to transfer data, while waiting for [interconnects](https://cloud.google.com/network-connectivity/docs/interconnect) to be provisioned.

Connectivity to additional on-prem sites or to other cloud providers should be implemented in a similar fashion, via VPN tunnels or interconnects, in the landing VPC (either trusted or untrusted, depending by the nature of the peers), sharing the same regional routers.

### Internal connectivity

Internal connectivity (e.g. between the trusted landing VPC and the spokes) is realized with VPC network peerings. As mentioned, there are other ways to implement connectivity. These can be easily retrofitted with minimal code changes, although they introduce additional considerations on service interoperability, quotas and management.

This is an options summary:

- [VPC Peering](https://cloud.google.com/vpc/docs/vpc-peering) (implemented here to connect the trusted landing VPC with the spokes)
  - Pros: no additional costs, full bandwidth with no configurations, no extra latency
  - Cons: no transitivity (e.g. to GKE masters, Cloud SQL, etc.), no selective exchange of routes, several quotas and limits shared between VPCs in a peering group
- [Multi-NIC appliances](https://cloud.google.com/architecture/best-practices-vpc-design#multi-nic) (implemented here to connect the trusted landing and untrusted VPCs)
  - Pros: provides additional security features (e.g. IPS), potentially better integration with on-prem systems by using the same vendor
  - Cons: complex HA/failover setup, limited by VM bandwidth and scale, additional costs for VMs and licenses, out of band management of a critical cloud component
- [VPN HA](https://cloud.google.com/network-connectivity/docs/vpn/concepts/topologies)
  - Pros: simple compatibility with GCP services that leverage peering internally, better control on routes, avoids peering groups shared quotas and limits
  - Cons: additional costs, marginal increase in latency, requires multiple tunnels for full bandwidth

### IP ranges, subnetting, routing

Minimizing the number of routes (and subnets) in the cloud environment is important, as it simplifies management and it avoids hitting [Cloud Router](https://cloud.google.com/network-connectivity/docs/router/quotas) and [VPC](https://cloud.google.com/vpc/docs/quota) quotas and limits. For this reason, we recommend to carefully plan the IP space used in your cloud environment. This allows the use of larger IP CIDR blocks in routes, whenever possible.

This stage uses a dedicated /16 block, which should be sized to the own needs. The subnets created in each VPC derive from this range.

Spoke VPCs also define and reserve two "special" CIDR ranges dedicated to [PSA (Private Service Access)](https://cloud.google.com/vpc/docs/private-services-access) and [Internal HTTPs Load Balancers (L7ILB)](https://cloud.google.com/load-balancing/docs/l7-internal).

Routes in GCP are either automatically created (for example, when a subnet is added to a VPC), manually created via static routes, dynamically exchanged through VPC peerings, or dynamically programmed by [Cloud Routers](https://cloud.google.com/network-connectivity/docs/router#docs) when a BGP session is established. BGP sessions can be configured to advertise VPC ranges, and/or custom ranges via custom advertisements.

In this setup:

- routes between multiple subnets within the same VPC are automatically exchanged by GCP
- the spokes and the trusted landing VPC exchange routes through VPC peerings
- on-premises is connected to the trusted landing VPC and it dynamically exchanges BGP routes with GCP (with the trusted VPC) using VPN HA
- for cross-environment (spokes) communications, and for connections to on-premises and to the Internet, the spokes leverage some default tagged routes that send the traffic of each region (whose machines are identified by a dedicated network tag, e.g. *ew1*) to a corresponding regional NVA in the trusted VPC, through an ILB (whose VIP is set as the route next-hop)
- the spokes are configured with backup default routes, so if the NVAs in the same region become unavailable, more routes to the NVAs in the other region are already available. Current routes are not able to understand if the next-hop ILBs become unhealthy. As such, in case of a regional failure, users will need to manually withdraw the primary default routes, so the secondaries will take over
- the NVAs are configured with static routes that allow the communication with on-premises and between the GCP resources (including the cross-environment communication)

The Cloud Routers (connected to the VPN gateways in the trusted VPC) are configured to exclude the default advertisement of VPC ranges and they only advertise their respective aggregate ranges, via custom advertisements. This greatly simplifies the routing configuration and avoids quota or limit issues, by keeping the number of routes small, instead of making it proportional to the subnets and to the secondary ranges in the VPCs.

### Internet egress

In this setup, Internet egress is realized through [Cloud NAT](https://cloud.google.com/nat/docs/overview), deployed in the untrusted landing VPC. This allows instances in all other VPCs to reach the Internet, passing through the NVAs (being the public Internet considered untrusted).

Several other scenarios are possible, with various degrees of complexity:

- deploy Cloud NAT in every VPC
- add forwarding proxies, with optional URL filters
- send Internet traffic to on-premises, so the existing egress infrastructure can be leveraged

Future pluggable modules will allow users to easily experiment with the above scenarios.

### VPC and Hierarchical Firewall

The GCP Firewall is a stateful, distributed feature that allows the creation of L4 policies, either via VPC-level rules or -more recently- via hierarchical policies, applied on the resource hierarchy (organization, folders).

The current setup adopts both firewall types. Hierarchical firewall rules are applied in the networking folder for common ingress rules (egress is open by default): for example, it allows the health checks and the IAP forwarders traffic to reach the VMs.

Rules and policies are defined in simple YAML files, described below.

### DNS

DNS often goes hand in hand with networking, especially on GCP where Cloud DNS zones and policies are associated at VPC level. This setup implements two DNS flows:

- on-premises to cloud, using private DNS zones pointing cloud-managed domains, and an [inbound policy](https://cloud.google.com/dns/docs/server-policies-overview#dns-server-policy-in) used as the forwarding target
- cloud to on-premises, leveraging Cloud DNS forwarding zones, pointing to the on-premise managed domains

The DNS configuration is centralized by leveraging peering zones, so that

- the landing project hosts the Cloud DNS configurations for the on-premise forwarding and Google API domains. Both the trusted and the untrusted VPCs are given visibility to these zones and the spokes consume them through their DNS peering zones
- Cloud DNS peering zones in the spokes host the environment-specific domains configurations, with the trusted and the untrusted VPCs acting as the consumers (leveraging the DNS peering zones configured in the landing project)

To complete the configuration, the 35.199.192.0/19 range should be routed to the VPN tunnels from on-premises, and the following names should be configured for DNS forwarding to cloud:

- `private.googleapis.com`
- `restricted.googleapis.com`
- `gcp.example.com` (used as a placeholder)

In GCP, a forwarding zone in the landing project is configured to forward queries to the placeholder domain `onprem.example.com` to on-premises.

This configuration is battle-tested, and flexible enough to lend itself to simple modifications without subverting its design.

## How to run this stage

This stage is meant to be executed after the [resman](../01-resman) stage has run. It leverages the automation service account and the storage bucket created there, and additional resources configured in the [bootstrap](../00-bootstrap) stage.

It's possible to run this stage in isolation, but that's outside of the scope of this document. Please, refer to the previous stages for the environment requirements.

Before running this stage, you need to make sure you have the correct credentials and permissions. You'll also need identify the module variables and make sure you assign them the values that match your configuration.

### Providers configuration

The default way of making sure you have the right permissions, is to use the identity of the service account pre-created for this stage, during the [resource management](../01-resman) stage, and that you are a member of the group that can impersonate it via provider-level configuration (`gcp-devops` or `organization-admins`).

To simplify the setup, the previous stage pre-configures a valid providers file in its output and optionally writes it to a local file if the `outputs_location` variable is set to a valid path.

If you have set a valid value for `outputs_location` in the bootstrap stage, simply link the relevant `providers.tf` file from this stage folder in the path you selected:

```bash
# `outputs_location` is set to `../../configs/example`
ln -s ../../configs/example/02-networking/providers.tf
```

If you have not configured `outputs_location` in bootstrap, you can derive the providers file from that stage outputs:

```bash
cd ../00-bootstrap
terraform output -json providers | jq -r '.["02-networking"]' \
  > ../02-networking-nva/providers.tf
```

### Variable configuration

There are two broad sets of variables you will need to fill in:

- variables shared by other stages (org id, billing account id, etc.), or derived from a resource managed by a different stage (folder id, automation project id, etc.)
- variables specific to resources managed by this stage

To avoid the tedious job of filling in the first group of variables with values derived from other stages outputs, the same mechanism used above for the provider configuration can be used to leverage pre-configured `.tfvars` files.

If you have set a valid value for `outputs_location` in the bootstrap and in the resman stage, simply link the relevant `terraform-*.auto.tfvars.json` files from this stage's folder in the path you specified, where the `*` above is set to the name of the stage that produced it. For this stage, a single `.tfvars` file is available:

```bash
# `outputs_location` is set to `../../configs/example`
ln -s ../../configs/example/02-networking/terraform-bootstrap.auto.tfvars.json
ln -s ../../configs/example/02-networking/terraform-resman.auto.tfvars.json
```
If you want to continue to rely on `outputs_location` logic, create a `terraform.tfvars` file and configure it as described [here](../00-bootstrap/#output-files-and-cross-stage-variables).

Please, refer to the [variables](#variables) table below for a map of the variable origins, and use the sections below to understand how to adapt this stage to your networking configuration.

### VPCs

VPCs are defined in separate files, one for `landing` (trusted and untrusted), one for `prod` and one for `dev`.

These files contain different resources:

- **project** ([`projects`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/project)): the "[host projects](https://cloud.google.com/vpc/docs/shared-vpc)" containing the VPCs and enabling the required APIs.
- **VPCs** ([`net-vpc`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc)): manage the subnets, the explicit routes for `{private,restricted}.googleapis.com` and the DNS inbound policy (for the trusted landing VPC). Subnets are created leveraging "resource factories": the configuration is separated from the module that implements it, and stored in a well-structured file. To add a new subnet, simply create a new file in the `data_folder` directory defined in the module, following the examples found in the [Fabric `net-vpc` documentation](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc#subnet-factory). Sample subnets are shipped in [data/subnets](./data/subnets) and can be easily customized to fit users' needs.

Subnets for [L7 ILBs](https://cloud.google.com/load-balancing/docs/l7-internal/proxy-only-subnets) are handled differently, and defined in variable `l7ilb_subnets`, while ranges for [PSA](https://cloud.google.com/vpc/docs/configure-private-services-access#allocating-range) are configured by variable `psa_ranges` - such variables are consumed by spoke VPCs.

- **Cloud NAT** ([`net-cloudnat`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-cloudnat)) (in the untrusted landing VPC only): it manages the networking infrastructure required to enable the Internet egress.

### VPNs

The connectivity between on-premises and GCP (the trusted landing VPC) is implemented with Cloud VPN HA ([`net-vpn`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpn)) and defined in [`vpn-onprem.tf`](./vpn-onprem.tf). The file implements a single logical connection between on-premises and the trusted landing VPC, both in `europe-west1` and `europe-west4`. The relevant parameters for its configuration are found in the variable `vpn_onprem_configs`.

### Routing and BGP

Each VPC network ([`net-vpc`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc)) manages a separate routing table, which can define static routes (e.g. to private.googleapis.com) and receives dynamic routes through VPC peering and BGP sessions established with the neighbor networks (e.g. the trusted landing VPC receives routes from on-premises, and the spokes receive RFC1918 from the trusted landing VPC).

Static routes are defined in `vpc-*.tf` files in the `routes` section of each `net-vpc` module.

BGP sessions for trusted landing to on-premises are configured through the variable `vpn_onprem_configs`.

### Firewall

**VPC firewall rules** ([`net-vpc-firewall`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc-firewall)) are defined per-vpc on each `vpc-*.tf` file and leverage a resource factory to massively create rules.
To add a new firewall rule, create a new file or edit an existing one in the `data_folder` directory defined in the module `net-vpc-firewall`, following the examples of the "[Rules factory](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc-firewall#rules-factory)" section of the module documentation. Sample firewall rules are shipped in [data/firewall-rules/landing-untrusted](./data/firewall-rules/landing-untrusted) and in [data/firewall-rules/landing-trusted](./data/firewall-rules/landing-trusted), and can be easily customized.

**Hierarchical firewall policies** ([`folder`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/folder)) are defined in `main.tf`, and managed through a policy factory implemented by the `folder` module, which applies the defined hierarchical to the `Networking` folder, which contains all the core networking infrastructure. Policies are defined in the `rules_file` file - to define a new one simply use the instructions found on "[Firewall policy factory](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/organization#firewall-policy-factory)". Sample hierarchical firewall policies are shipped in [data/hierarchical-policy-rules.yaml](./data/hierarchical-policy-rules.yaml) and can be easily customized.

### DNS architecture

The DNS ([`dns`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/dns)) infrastructure is defined in [`dns-*.tf`] files.

Cloud DNS manages on-premises forwarding, the main GCP zone (in this example `gcp.example.com`) and is peered to environment-specific zones (i.e. `dev.gcp.example.com` and `prod.gcp.example.com`).

#### Cloud environment

The root DNS zone defined in the landing project acts as the source of truth for DNS within the Cloud environment. The resources defined in the spoke VPCs consume the landing DNS infrastructure through DNS peering (e.g. `prod-landing-root-dns-peering`).
The spokes can optionally define private zones (e.g. `prod-dns-private-zone`). Granting visibility both to the trusted and untrusted landing VPCs ensures that the whole cloud environment can query such zones.

#### Cloud to on-premises

Leveraging the forwarding zone defined in the landing project (e.g. `onprem-example-dns-forwarding` and `reverse-10-dns-forwarding`), the cloud environment can resolve `in-addr.arpa.` and `onprem.example.com.` using the on-premise DNS infrastructure. On-premise resolver IPs are set in the variable `dns.onprem`.

DNS queries sent to the on-premise infrastructure come from the `35.199.192.0/19` source range.

#### On-premises to cloud

The [Inbound DNS Policy](https://cloud.google.com/dns/docs/server-policies-overview#dns-server-policy-in) defined in the *trusted landing VPC module* ([`vpc-landing.tf`](./vpc-landing.tf)) automatically reserves the first available IP address on each subnet (typically the third one in a CIDR) to expose the Cloud DNS service, so that it can be consumed from outside of GCP.

### Private Google Access

[Private Google Access](https://cloud.google.com/vpc/docs/private-google-access) (or PGA) is configured in this environment. It enables VMs and on-premise systems to consume Google APIs from within the Google network.

For PGA to work:

- Private Google Access should be enabled on the subnet. \
Subnets created using the `net-vpc` module are PGA-enabled by default.

- 199.36.153.4/30 (`restricted.googleapis.com`) and 199.36.153.8/30 (`private.googleapis.com`) should be routed from on-premises to the trusted landing VPC, and from there to the `default-internet-gateway`. \
The `vpn_onprem_configs` variable contains the ranges advertised from GCP to on-premises. Furthermore, the trusted landing VPC (e.g. see `landing-trusted-vpc` in [`vpc-landing.tf`](./vpc-landing.tf)) has explicit routes to send traffic destined to restricted and private - googleapis.com to the Internet gateway (which works for Google APIs only, and not for the whole Internet, since Cloud NAT is not configured in the trusted landing VPC).

- On-premises, a private DNS zone for `googleapis.com` should be created and configured per [this article](https://cloud.google.com/vpc/docs/configure-private-google-access-hybrid#config-domain). Its configuration can be copied from the module `googleapis-private-zone` in [`dns.tf`](./dns.tf)

### Preliminar activities

Before running `terraform apply`, make sure to adapt `variables.tf` to your needs, to update the variable values using a new `terraform.tfvars` file, and to update the references to the regions in the whole directory, in order to match your preferences (e.g. `europe-west1` or `ew1`).

If you're not using other FAST stages, you'll also need to create a `providers.tf` file to configure the GCS backend and the service account to use to run the deployment.

You're now ready to run `terraform init` and `terraform apply`.

### Post-deployment activities

- On-premise routers should be configured to advertise all relevant CIDRs to the GCP environments. To avoid hitting GCP quotas, we recommend aggregating routes as much as possible
- On-premise routers should accept BGP sessions from their cloud peers
- On-premise DNS servers should have forward zones configured, in order to resolve GCP-managed domains

## Customizations

### Adding an environment

To create a new environment (e.g. `staging`), a few changes are required:

Create a `vpc-spoke-staging.tf` file by copying `vpc-spoke-prod.tf` file.
Adapt the new file by replacing the value "prod" with the value "staging".
Running `diff vpc-spoke-dev.tf vpc-spoke-prod.tf` can help to see how environment files differ.

The new VPC requires a set of dedicated CIDRs, one per region, added to variable `custom_adv` (for example as `spoke_staging_ew1` and `spoke_staging_ew4`).
>`custom_adv` is a map that "resolves" CIDR names to the actual addresses, and will be used later to configure routing.
>
Variables managing L7 Internal Load Balancers (`l7ilb_subnets`) and Private Service Access (`psa_ranges`) should also be adapted, and subnets and firewall rules for the new spoke should be added, as described above.

VPC network peering connectivity to the `trusted landing VPC` is managed by the `vpc-peering-*.tf` files.
Copy `vpc-peering-prod.tf` to `vpc-peering-staging.tf` and replace "prod" with "staging", where relevant.

Configure the NVAs deployed or update the sample [NVA config file](data/nva-startup-script.tftpl) making sure they support the new subnets.

DNS configurations are managed in the `dns-*.tf` files.
Copy the `dns-prod.tf` to `dns-staging.tf` and replace within the files "prod" with "staging", where relevant.
Don't forget to add a peering zone in the landing project and point it to the newly created environment private zone.

<!-- TFDOC OPTS files:1 show_extra:1 -->
<!-- BEGIN TFDOC -->

## Files

| name | description | modules | resources |
|---|---|---|---|
| [dns-dev.tf](./dns-dev.tf) | Development spoke DNS zones and peerings setup. | <code>dns</code> |  |
| [dns-landing.tf](./dns-landing.tf) | Landing DNS zones and peerings setup. | <code>dns</code> |  |
| [dns-prod.tf](./dns-prod.tf) | Production spoke DNS zones and peerings setup. | <code>dns</code> |  |
| [main.tf](./main.tf) | Networking folder and hierarchical policy. | <code>folder</code> |  |
| [monitoring.tf](./monitoring.tf) | Network monitoring dashboards. |  | <code>google_monitoring_dashboard</code> |
| [nva.tf](./nva.tf) | None | <code>compute-mig</code> · <code>compute-vm</code> · <code>net-ilb</code> |  |
| [outputs.tf](./outputs.tf) | Module outputs. |  | <code>local_file</code> |
| [test-resources.tf](./test-resources.tf) | temporary instances for testing | <code>compute-vm</code> |  |
| [variables.tf](./variables.tf) | Module variables. |  |  |
| [vpc-landing.tf](./vpc-landing.tf) | Landing VPC and related resources. | <code>net-cloudnat</code> · <code>net-vpc</code> · <code>net-vpc-firewall</code> · <code>project</code> |  |
| [vpc-spoke-dev.tf](./vpc-spoke-dev.tf) | Dev spoke VPC and related resources. | <code>net-address</code> · <code>net-vpc</code> · <code>net-vpc-firewall</code> · <code>net-vpc-peering</code> · <code>project</code> |  |
| [vpc-spoke-prod.tf](./vpc-spoke-prod.tf) | Production spoke VPC and related resources. | <code>net-address</code> · <code>net-vpc</code> · <code>net-vpc-firewall</code> · <code>net-vpc-peering</code> · <code>project</code> |  |
| [vpn-onprem.tf](./vpn-onprem.tf) | VPN between landing and onprem. | <code>net-vpn-ha</code> |  |

## Variables

| name | description | type | required | default | producer |
|---|---|:---:|:---:|:---:|:---:|
| [billing_account_id](variables.tf#L17) | Billing account id. | <code>string</code> | ✓ |  | <code>00-bootstrap</code> |
| [folder_ids](variables.tf#L59) | Folders to be used for the networking resources in folders/nnnnnnnnnnn format. If null, folder will be created. | <code>map&#40;string&#41;</code> | ✓ |  | <code>01-resman</code> |
| [organization](variables.tf#L91) | Organization details. | <code title="object&#40;&#123;&#10;  domain      &#61; string&#10;  id          &#61; number&#10;  customer_id &#61; string&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> | ✓ |  | <code>00-bootstrap</code> |
| [prefix](variables.tf#L107) | Prefix used for resources that need unique names. | <code>string</code> | ✓ |  | <code>00-bootstrap</code> |
| [custom_adv](variables.tf#L23) | Custom advertisement definitions in name => range format. | <code>map&#40;string&#41;</code> |  | <code title="&#123;&#10;  cloud_dns                 &#61; &#34;35.199.192.0&#47;19&#34;&#10;  gcp_all                   &#61; &#34;10.128.0.0&#47;16&#34;&#10;  gcp_dev_ew1               &#61; &#34;10.128.128.0&#47;19&#34;&#10;  gcp_dev_ew4               &#61; &#34;10.128.160.0&#47;19&#34;&#10;  gcp_landing_trusted_ew1   &#61; &#34;10.128.64.0&#47;19&#34;&#10;  gcp_landing_trusted_ew4   &#61; &#34;10.128.96.0&#47;19&#34;&#10;  gcp_landing_untrusted_ew1 &#61; &#34;10.128.0.0&#47;19&#34;&#10;  gcp_landing_untrusted_ew4 &#61; &#34;10.128.32.0&#47;19&#34;&#10;  gcp_prod_ew1              &#61; &#34;10.128.192.0&#47;19&#34;&#10;  gcp_prod_ew4              &#61; &#34;10.128.224.0&#47;19&#34;&#10;  googleapis_private        &#61; &#34;199.36.153.8&#47;30&#34;&#10;  googleapis_restricted     &#61; &#34;199.36.153.4&#47;30&#34;&#10;  rfc_1918_10               &#61; &#34;10.0.0.0&#47;8&#34;&#10;  rfc_1918_172              &#61; &#34;172.16.0.0&#47;12&#34;&#10;  rfc_1918_192              &#61; &#34;192.168.0.0&#47;16&#34;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| [data_dir](variables.tf#L45) | Relative path for the folder storing configuration data for network resources. | <code>string</code> |  | <code>&#34;data&#34;</code> |  |
| [dns](variables.tf#L51) | Onprem DNS resolvers | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code title="&#123;&#10;  onprem &#61; &#91;&#34;10.0.200.3&#34;&#93;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| [l7ilb_subnets](variables.tf#L65) | Subnets used for L7 ILBs. | <code title="map&#40;list&#40;object&#40;&#123;&#10;  ip_cidr_range &#61; string&#10;  region        &#61; string&#10;&#125;&#41;&#41;&#41;">map&#40;list&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;&#41;</code> |  | <code title="&#123;&#10;  prod &#61; &#91;&#10;    &#123; ip_cidr_range &#61; &#34;10.128.92.0&#47;24&#34;, region &#61; &#34;europe-west1&#34; &#125;,&#10;    &#123; ip_cidr_range &#61; &#34;10.128.93.0&#47;24&#34;, region &#61; &#34;europe-west4&#34; &#125;&#10;  &#93;&#10;  dev &#61; &#91;&#10;    &#123; ip_cidr_range &#61; &#34;10.128.60.0&#47;24&#34;, region &#61; &#34;europe-west1&#34; &#125;,&#10;    &#123; ip_cidr_range &#61; &#34;10.128.61.0&#47;24&#34;, region &#61; &#34;europe-west4&#34; &#125;&#10;  &#93;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| [onprem_cidr](variables.tf#L83) | Onprem addresses in name => range format. | <code>map&#40;string&#41;</code> |  | <code title="&#123;&#10;  main &#61; &#34;10.0.0.0&#47;24&#34;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| [outputs_location](variables.tf#L101) | Path where providers and tfvars files for the following stages are written. Leave empty to disable. | <code>string</code> |  | <code>null</code> |  |
| [project_factory_sa](variables.tf#L113) | IAM emails for project factory service accounts | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> | <code>01-resman</code> |
| [psa_ranges](variables.tf#L120) | IP ranges used for Private Service Access (e.g. CloudSQL). | <code>map&#40;map&#40;string&#41;&#41;</code> |  | <code title="&#123;&#10;  prod &#61; &#123;&#10;    cloudsql-mysql     &#61; &#34;10.128.94.0&#47;24&#34;&#10;    cloudsql-sqlserver &#61; &#34;10.128.95.0&#47;24&#34;&#10;  &#125;&#10;  dev &#61; &#123;&#10;    cloudsql-mysql     &#61; &#34;10.128.62.0&#47;24&#34;&#10;    cloudsql-sqlserver &#61; &#34;10.128.63.0&#47;24&#34;&#10;  &#125;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| [router_configs](variables.tf#L135) | Configurations for CRs and onprem routers. | <code title="map&#40;object&#40;&#123;&#10;  adv &#61; object&#40;&#123;&#10;    custom  &#61; list&#40;string&#41;&#10;    default &#61; bool&#10;  &#125;&#41;&#10;  asn &#61; number&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code title="&#123;&#10;  landing-trusted-ew1 &#61; &#123;&#10;    asn &#61; &#34;65534&#34;&#10;    adv &#61; null&#10;  &#125;&#10;  landing-trusted-ew4 &#61; &#123;&#10;    asn &#61; &#34;65534&#34;&#10;    adv &#61; null&#10;  &#125;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| [vpn_onprem_configs](variables.tf#L158) | VPN gateway configuration for onprem interconnection. | <code title="map&#40;object&#40;&#123;&#10;  adv &#61; object&#40;&#123;&#10;    default &#61; bool&#10;    custom  &#61; list&#40;string&#41;&#10;  &#125;&#41;&#10;  peer_external_gateway &#61; object&#40;&#123;&#10;    redundancy_type &#61; string&#10;    interfaces &#61; list&#40;object&#40;&#123;&#10;      id         &#61; number&#10;      ip_address &#61; string&#10;    &#125;&#41;&#41;&#10;  &#125;&#41;&#10;  tunnels &#61; list&#40;object&#40;&#123;&#10;    peer_asn                        &#61; number&#10;    peer_external_gateway_interface &#61; number&#10;    secret                          &#61; string&#10;    session_range                   &#61; string&#10;    vpn_gateway_interface           &#61; number&#10;  &#125;&#41;&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code title="&#123;&#10;  landing-trusted-ew1 &#61; &#123;&#10;    adv &#61; &#123;&#10;      default &#61; false&#10;      custom &#61; &#91;&#10;        &#34;cloud_dns&#34;, &#34;googleapis_private&#34;, &#34;googleapis_restricted&#34;, &#34;gcp_all&#34;&#10;      &#93;&#10;    &#125;&#10;    peer_external_gateway &#61; &#123;&#10;      redundancy_type &#61; &#34;SINGLE_IP_INTERNALLY_REDUNDANT&#34;&#10;      interfaces &#61; &#91;&#10;        &#123; id &#61; 0, ip_address &#61; &#34;8.8.8.8&#34; &#125;,&#10;      &#93;&#10;    &#125;&#10;    tunnels &#61; &#91;&#10;      &#123;&#10;        peer_asn                        &#61; 65534&#10;        peer_external_gateway_interface &#61; 0&#10;        secret                          &#61; &#34;foobar&#34;&#10;        session_range                   &#61; &#34;169.254.1.0&#47;30&#34;&#10;        vpn_gateway_interface           &#61; 0&#10;      &#125;,&#10;      &#123;&#10;        peer_asn                        &#61; 65534&#10;        peer_external_gateway_interface &#61; 0&#10;        secret                          &#61; &#34;foobar&#34;&#10;        session_range                   &#61; &#34;169.254.1.4&#47;30&#34;&#10;        vpn_gateway_interface           &#61; 1&#10;      &#125;&#10;    &#93;&#10;  &#125;&#10;  landing-trusted-ew4 &#61; &#123;&#10;    adv &#61; &#123;&#10;      default &#61; false&#10;      custom &#61; &#91;&#10;        &#34;cloud_dns&#34;, &#34;googleapis_private&#34;, &#34;googleapis_restricted&#34;, &#34;gcp_all&#34;&#10;      &#93;&#10;    &#125;&#10;    peer_external_gateway &#61; &#123;&#10;      redundancy_type &#61; &#34;SINGLE_IP_INTERNALLY_REDUNDANT&#34;&#10;      interfaces &#61; &#91;&#10;        &#123; id &#61; 0, ip_address &#61; &#34;8.8.8.8&#34; &#125;,&#10;      &#93;&#10;    &#125;&#10;    tunnels &#61; &#91;&#10;      &#123;&#10;        peer_asn                        &#61; 65534&#10;        peer_external_gateway_interface &#61; 0&#10;        secret                          &#61; &#34;foobar&#34;&#10;        session_range                   &#61; &#34;169.254.1.0&#47;30&#34;&#10;        vpn_gateway_interface           &#61; 0&#10;      &#125;,&#10;      &#123;&#10;        peer_asn                        &#61; 65534&#10;        peer_external_gateway_interface &#61; 0&#10;        secret                          &#61; &#34;foobar&#34;&#10;        session_range                   &#61; &#34;169.254.1.4&#47;30&#34;&#10;        vpn_gateway_interface           &#61; 1&#10;      &#125;&#10;    &#93;&#10;  &#125;&#10;&#125;">&#123;&#8230;&#125;</code> |  |

## Outputs

| name | description | sensitive | consumers |
|---|---|:---:|---|
| [project_ids](outputs.tf#L42) | Network project ids. |  |  |
| [project_numbers](outputs.tf#L51) | Network project numbers. |  |  |
| [shared_vpc_host_projects](outputs.tf#L60) | Shared VPC host projects. |  |  |
| [shared_vpc_self_links](outputs.tf#L69) | Shared VPC host projects. |  |  |
| [tfvars](outputs.tf#L93) | Network-related variables used in other stages. | ✓ |  |
| [vpn_gateway_endpoints](outputs.tf#L79) | External IP Addresses for the GCP VPN gateways. |  |  |

<!-- END TFDOC -->