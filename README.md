# gateway-load-balancer-demo-cloudformation

CloudFormation templates implementing AWS's **centralized inspection
architecture**: a Gateway Load Balancer (GWLB) fronting firewall/appliance
instances, with a Transit Gateway routing traffic from spoke VPCs through
that inspection point.

## Attribution

`appliance-vpc-gwlb-firewalls.yaml` and `transit-gateway.yaml` are adapted
from AWS's own published reference architecture - see
[Centralized inspection architecture with AWS Gateway Load Balancer and AWS Transit Gateway](https://aws.amazon.com/blogs/networking-and-content-delivery/centralized-inspection-architecture-with-aws-gateway-load-balancer-and-aws-transit-gateway/)
(the original blog post has an architecture diagram worth reading alongside
this repo). `spoke-vpc-3az-baseline.yaml` is a separate, more general-purpose
3-AZ VPC baseline (public/private/internal tiers + optional Flow Logs and
VPC endpoints) used here as an example spoke VPC - it isn't from the AWS
sample series and doesn't itself create the TGW attachment described below.

## Architecture

```
Spoke VPC 1 --\                                    +------------------+
               >-- Transit Gateway ---------------> | Appliance VPC    |
Spoke VPC 2 --/     (Egress RTB: default->Appliance)|                  |
                     (Transit RTB: Spoke CIDRs)     |  GWLB (GENEVE)   |
                                                     |   |         |    |
                                                     | Appliance1 Appliance2
                                                     | (AZ 1)     (AZ 2) |
                                                     +------------------+
```

The appliance instances run a demo HTTP server and `iptables` rules that
hairpin GENEVE traffic (UDP/6081) back to the GWLB - **the iptables policy
is `ACCEPT` for everything**, so no real filtering happens. This is
intentional per AWS's own sample: it's a wiring/routing demo, not a
firewall product.

## Deploy order

1. **`appliance-vpc-gwlb-firewalls.yaml`** - creates the Appliance VPC, the
   2 firewall appliance instances, the GWLB, its VPC Endpoint Service, and
   2 GWLB VPC endpoints (one per AZ). Needs an existing EC2 KeyPair name.
2. **`transit-gateway.yaml`** - creates the Transit Gateway and wires it to
   the Appliance VPC. Feed it these parameters from stack 1's outputs:

   | This stack's parameter | From stack 1's output |
   |---|---|
   | `ApplianceVpcId` | `ApplianceVpcId` |
   | `ApplianceVpcTgwAttachSubnet1Id` | `ApplianceTgwAttachSubnet1Id` |
   | `ApplianceVpcTgwAttachSubnet2Id` | `ApplianceTgwAttachSubnet2Id` |
   | `ApplianceVpcApplianceRtb1Id` | `ApplianceRtb1Id` |
   | `ApplianceVpcApplianceRtb2Id` | `ApplianceRtb2Id` |

   Also pass `Spoke1VpcCidr`/`Spoke2VpcCidr` (must match whatever spoke
   VPC CIDRs you actually use) and `AccessLocation`.
3. **`spoke-vpc-3az-baseline.yaml`** (deploy twice, once per spoke) - a
   standalone 3-AZ VPC. This template does **not** create the TGW
   attachment itself; you still need to attach each spoke VPC to the
   Transit Gateway created in step 2 (e.g. `AWS::EC2::TransitGatewayVpcAttachment`)
   and associate it with the Egress route table for traffic to actually
   flow through the inspection path.

## Prerequisites

- An existing EC2 KeyPair in the target region (`KeyPairName` parameter).
- AWS CLI, IAM permissions for `CAPABILITY_NAMED_IAM` (appliance instance
  roles) and Lambda (the custom resource that looks up the GWLB's VPC
  Endpoint Service name).

## Cost warning

This demo is not cheap to leave running: GWLB (hourly + per-GB processed),
2 NAT Gateways, 2 EC2 appliance instances, Transit Gateway (hourly per
attachment + per-GB), plus whatever the spoke VPCs add. Tear down when not
actively testing.

## Security notes - this is a POC, not a hardening reference

- `AccessLocation` (SSH source CIDR) defaults to `0.0.0.0/0` - the
  template's own parameter description calls this out as "not
  recommended." Restrict it to your IP before deploying.
- The appliance instances' `iptables` rules accept all traffic (see
  Architecture above) - no actual inspection/filtering happens; this is a
  routing/wiring demo only.
- `spoke-vpc-3az-baseline.yaml`'s NACLs allow all traffic by default
  (annotated with `cfn_nag` suppressions acknowledging this), and its S3
  Gateway Endpoint policy is unrestricted (`Principal: "*"`). Fine for a
  POC baseline, not for production as-is.

## Verifying it works

After all 3 stacks are up and a spoke VPC is attached to the TGW, traffic
from an instance in the spoke VPC to the internet (or to the other spoke)
routes through the Transit Gateway's Egress route table into the Appliance
VPC, gets GENEVE-encapsulated to the GWLB, hairpinned back out through
whichever appliance is healthy, and returns via the Transit route table.
`curl`-ing the appliance's own demo page directly (from within the
Appliance VPC) confirms the instance itself is reachable and serving
traffic.

## Files

| File | Contents |
|---|---|
| `appliance-vpc-gwlb-firewalls.yaml` | Appliance VPC, GWLB, 2 appliance EC2 instances, VPC Endpoint Service + endpoints |
| `transit-gateway.yaml` | Transit Gateway, TGW attachment to the Appliance VPC, Egress/Transit route tables |
| `spoke-vpc-3az-baseline.yaml` | Standalone 3-AZ spoke VPC baseline (public/private/internal tiers, optional Flow Logs + VPC endpoints) |
