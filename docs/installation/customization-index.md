# 1. AWS Secure Environment Accelerator

- [1. AWS Secure Environment Accelerator](#1-aws-secure-environment-accelerator)
  - [1.1. **Sample Accelerator Configuration Files**](#11-sample-accelerator-configuration-files)
  - [1.2. **Deployment Customizations**](#12-deployment-customizations)
  - [1.3. Other Configuration File Hints and Tips](#13-other-configuration-file-hints-and-tips)

## 1.1. **Sample Accelerator Configuration Files**

- Full PBMM configuration [file](../../reference-artifacts/config.example.json)
  - The full PBMM configuration file was based on feedback from customers moving into AWS at scale and at a rapid pace. Customers of this nature have indicated that they do not want to have to upsize their perimeter firewalls or add Interface endpoints as their developers start to use new AWS services. These are the two most expensive components of the solution.
- Light weight PBMM configuration [file](../../reference-artifacts/config.lite-example.json) **(Recommended for most new PBMM customers)**
  - To reduce solution costs and allow customers to grow into more advanced AWS capabilities, we created this lighter weight configuration that does not sacrifice functionality, but could limit performance. This config file:
    - only deploys the 6 required centralized Interface Endpoints (removes 56). All services remain accessible using the AWS public endpoints, but require traversing the perimeter firewalls
    - removes the perimeter VPC Interface Endpoints
    - reduces the Fortigate instance sizes from c5n.2xl to c5n.xl (VM08 to VM04)
    - removes the Unclass ou and VPC
  - The Accelerator allows customers to easily add or change this functionality in future, as and when required without any impact

## 1.2. **Deployment Customizations**

- The sample configuration files are provided as single, all encompassing, json files. The Accelerator also supports both splitting the config file into multiple component files and configuration files built using YAML instead of json. This is documented [here](./multi-file-config-capabilities.md)

- The sample configuration files do not include the full range of supported configuration file parameters and values, additional configuration file parameters and values can be found [here](../../reference-artifacts/config-sample-snippets/sample_snippets.md)

- The Accelerator is provided with a sample 3rd party configuration file to demonstrate automated deployment of 3rd party firewall technologies. Given the code is vendor agnostic, this process should be able to be leveraged to deploy other vendors firewall appliances. When and if other options become available, we will add them here as well.
  - Automated firewall configuration [customization](../../reference-artifacts/config-sample-snippets/firewall_file_available_variables.md) possibilities
  - Sample Fortinet Fortigate firewall config [file](../../reference-artifacts/Third-Party/firewall-example.txt)

## 1.3. Other Configuration File Hints and Tips

- You cannot supply (or change) configuration file values to something not supported by the AWS platform
  - For example, CWL retention only supports specific retention values (not any number)
  - Shard count - can only increase/reduce by half the current limit. i.e. you can change from `1`-`2`, `2`-`3`, `4`-`6`
- Always add any new items to the END of all lists or sections in the config file, otherwise
  - Update validation checks will fail (VPC's, subnets, share-to, etc.)
  - VPC endpoint deployments will fail - do NOT re-order or insert VPC endpoints (unless you first remove them all completely, execute the state machine, then re-add them, and again run the state machine) - this challenge no longer exists as of v1.2.1.
- To skip, remove or uninstall a component, you can often simply change the section header, instead of removing the section
  - change "deployments"/"firewalls" to "deployments"/"xxfirewalls" and it will uninstall the firewalls and maintain the old config file settings for future use
  - Objects with the parameter deploy: true, support setting the value to false to remove the deployment
- As you grow and add AWS accounts, the Kinesis Data stream in the log-archive account will need to be monitored and have its capacity (shard count) increased by setting `"kinesis-stream-shard-count"` variable under `"central-log-services"` in the config file
- Updates to NACL's requires changing the rule number (`100` to `101`) or they will fail to update
- When adding a new subnet or subnets to a VPC (including enabling an additional AZ), you need to:
  - increment any impacted nacl id's in the config file (`100` to `101`, `32000` to `32001`) (CFN does not allow nacl updates)
  - make a minor change to any impacted route table names (`MyRouteTable` to `MyRouteTable1`) (CFN does not allow updates to route table associated ids)
  - prior to v1.2.3, if adding a subnet that is associated with the TGW, you need to remove the TGW association (`"tgw-attach"` to `"xxtgw-attach"` for the VPC) and then re-attach on a subsequent state machine execution. This is resolved in v1.2.3.
- The sample firewall configuration uses an instance with **4** NIC's, make sure you use an instance size that supports 4 ENI's
- Re-enabling individual security controls in Security Hub requires toggling the entire security standard off and on again, controls can be disabled at any time
- Firewall names, CGW names, TGW names, MAD Directory ID, account keys, and ou's must all be unique throughout the entire configuration file (also true for VPC names given nacl and security group referencing design)
- The configuration file _does_ have validation checks in place that prevent users from making certain major unsupported configuration changes
- **The configuration file does _NOT_ have extensive error checking. It is expected you know what you are doing. We eventually hope to offer a config file, wizard based GUI editor and add the validation logic in this separate tool. In most cases the State Machine will fail with an error, and you will simply need to troubleshoot, rectify and rerun the state machine.**
- You cannot move an account between top-level ou's. This would be a security violation and cause other issues. You can move accounts between sub-ou. Note: The ALZ version of the Accelerator does not support sub-ou.
- v1.1.5 and above adds support for customer provided YAML config file(s) as well as JSON. In future we will be providing a version of the config file with comments describing the purpose of each configuration item
- Security Group names were designed to be identical between environments, if you want the VPC name in the SG name, you need to do it manually in the config file
- We only support the subset of yaml that converts to JSON (we do not support anchors)
- We do NOT support changing the `organization-admin-role`, this value must be set to `AWSCloudFormationStackSetExecutionRole` at this time.
- Adding more than approximately 50 _new_ VPC Interface Endpoints across _all_ regions in any one account in any single state machine execution will cause the state machine to fail due to Route 53 throttling errors. If adding endpoints at scale, only deploy 1 region at a time. In this scenario, the stack(s) will fail to properly delete, also based on the throttling, and will require manual removal.
- If `use-central-endpoints` is changed from true to false, you cannot add a local vpc endpoint on the same state machine execution (add the endpoint on a prior or subsequent execution)
  - in versions 1.2.0 through 1.2.2 there is a issue adding local endpoints when a central endpoint already exists for the vpc
- If you update the firewall names, be sure to update the routes and alb's which point to them. Firewall licensing occurs through the management port, which requires a VPC route back to the firewall to get internet access and validate the firewall license.
- Initial MAD deployments are only supported in 2 AZ subnets (as of v1.2.3). Deploy the Accelerator with only 2 MAD subnets and add additional AZ's on subsequent state machine executions. A fix is planned.
- In v1.2.3 and below (fixes planned for v1.2.4):
  - if the same IAM policy file is used in more than one spot in the config, we require one account to reference the policy twice or you will get a `Unexpected token u in JSON at position 0,` error in Phase 1
  - the `zones\resolver-vpc` is a mandatory parameter, you must deploy a small dummy vpc w/no subnets, routes, etc. in the account of your choosing for this validation to succeed
  - security hub deploys security standards and disables controls, no automated mechanism exists to disable security standard or re-enable individual controls

---

[...Return to Accelerator Table of Contents](../index.md)
