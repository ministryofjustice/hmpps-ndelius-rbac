# NDelius Role-Based Access Control

This repository contains the static LDAP data required to support the National Delius application and includes schemas, service users (API clients), roles and groups.

## Roles
Roles define access to a single area of functionality within Delius, and are made up of a collection of allowed business interactions.
 
New roles should be added to [nd_role_catalogue.ldif](roles/nd_role_catalogue.ldif), under the `cn=ndRoleCatalogue` container.
They generally follow the naming format of 
```
cn: {XXX}BT{NNN}
```
where:
* `{XXX}` is an acronym corresponding to the functional area that the role relates to. For example, `OI` for Offender Index.
* `BT` stands for *Business Transaction*.
* `{NNN}` is the numeric identifier of the role within the functional area, starting from 001.

Role definitions should contain a meaningful `description` attribute, as well as:
* `sector` - Whether this role should be limited to `public` (NPS) users or `private` (CRC) users. Omit this attribute if the role can be applied to both sets of users.
* `adminlevel` - Whether this role should be limited to `national` or `local` administrators. Omit this attribute if the role can be applied to both sets of users.
* `uibusinessinteractioncollection` - A list of business interaction codes, used for auditing and securing specific interactions within NDelius. Generally follows the naming format of `{XXX}BI{NNN}`.
* `level1` - Boolean defining whether the role relates to functionality accessible to the *current* offender manager.
* `level2` - Boolean defining whether the role relates to functionality accessible to any *previous* offender manager.
* `level3` - Boolean defining whether the role relates to functionality accessible to the manager of an offender's sub-component (eg. Event, NSI etc).
* `objectclass` - Every role should have an objectclass of `NDRole` as well as `top`.

## Role Groups
Role Groups define a simple collection of Roles, intended for use by User Administrators as a convenient way of applying a predefined set of roles to users. 
Note: Only the roles themselves are stored against the user, information about which role groups were applied is not persisted.

New role groups should be added to [nd_role_groups.ldif](roles/nd_role_groups.ldif), under the `cn=ndRoleGroups` container.
They should specify an `objectclass: NDRoleGroup` and contain a user-facing description in the `cn` attribute.

Roles within a role group should be defined as child records of the role group, with `objectclass: NDRoleAssociation` and an `aliasedObjectName` attribute pointing to the distinguished name of the role in the catalogue.

## Service Users & Clients
Service users should correspond to *external interfacing systems*, that make calls to NDelius API endpoints in some way.

New service users should be added to [service-users.ldif.j2](users/service-users.ldif.j2) under the `cn=EISUsers` container, with `objectclass: NDUser`.
Optionally, you can also add `objectclass: NDClient` which will mark the user as an OAuth client for use in authenticating with NDelius using the [User Management Tool](https://github.com/ministryofjustice/ndelius-um#authentication).
OAuth clients should also provide `authorizedGrantType`, `registeredRedirectUri` and `resourceId` attributes.

### Assigning roles
Historically, roles have been assigned to service users by setting the allowed interactions assigned to them in the `businesscategory` attribute.
See for example the definitions of the OASYS, IAPS and DSS users in [service-users.ldif.j2](users/service-users.ldif.j2).

For newer service users (eg. CRCs), the roles are assigned in the same way as with NDelius users.
That is, by adding child entries with `objectclass: NDRoleAssociation`. 
These entries act as aliases for the associated `NDRole` object in the role catalogue.

### Setting a secure password
Passwords should be randomly generated per-user per-environment and stored in the [AWS SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html).
This can be done automatically for Delius by specifying the parameter details in the [hmpps-env-configs](https://github.com/ministryofjustice/hmpps-env-configs) repo (See [common.credentials.yml](https://github.com/ministryofjustice/hmpps-env-configs/blob/master/common/common.credentials.yml)).
Once any new parameters have been added to the credentials files, the values are generated and loaded into the environment-specific parameter store by running the [Create Credentials](https://jenkins.engineering-dev.probation.hmpps.dsd.io/job/DAMS/job/Environments/job/delius-core-dev/job/Security/job/Create_credentials/) job in Jenkins.

After a password has been generated in the parameter store, it can be referenced in this repo by using the `aws_ssm` lookup. For example:
```jinja2
{% set my_password=lookup('aws_ssm', 'paramater-name', region=region) %}
```
These parameters are securely fetched at runtime during RBAC deployment.

## Deployment
To deploy a release of this repo into a Delius environment, you should first update the [environment configuration repository](https://github.com/ministryofjustice/hmpps-env-configs) with the desired version.
As an example, for the `dev` environment you should update the value for `rbac_version` in [delius-core-dev/ansible/group_vars/ldap.yml](https://github.com/ministryofjustice/hmpps-env-configs/blob/master/delius-core-dev/ansible/group_vars/ldap.yml).
A pull request must be created for the change which should be peer-reviewed by the [hmpps-devops](https://github.com/orgs/ministryofjustice/teams/hmpps-devops) team before merging to master.

The release can then be deployed to the environment using the [Deploy Application](https://jenkins.engineering-dev.probation.hmpps.dsd.io/job/DAMS/job/Environments/job/delius-core-dev/job/Delius/job/Deploy%20application/) job in Jenkins, making sure that the `Uplift LDAP` parameter is enabled.

## Contact
* For technical information about this repository, please contact the Delius Infrastructure Support team via the [#delius_infra_support](https://mojdt.slack.com/archives/CNXK9893K) Slack channel.
* For domain-specific information about roles and groups in Delius, contact the National Delius Service Team (NDST) via the [#ndst-all](https://mojdt.slack.com/archives/GEAGES37Y) Slack channel.
