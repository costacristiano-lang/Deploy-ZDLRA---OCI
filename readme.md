# Oracle Database Autonomous Recovery Service on OCI

## Deployment and Configuration Guide

> This implementation guide is based on Oracle documentation and is intended for customer use. It is not an Oracle-issued document and must be validated against the current Oracle documentation and the customer's change-management process.

[English](./README-oci-zero-data-loss-recovery-service.md) | [Português (Brasil)](./README-oci-zero-data-loss-recovery-service.pt-BR.md)

This README provides a baseline procedure for deploying and configuring **Oracle Database Zero Data Loss Autonomous Recovery Service** on Oracle Cloud Infrastructure (OCI), protecting Oracle databases in OCI, Exadata Database Service, and supported multicloud environments.

> In OCI, the managed service is called **Database Autonomous Recovery Service** or **Oracle Database Zero Data Loss Autonomous Recovery Service**. **ZDLRA** usually refers to Oracle Zero Data Loss Recovery Appliance, the appliance/engineered system. For OCI, use this guide as a reference for the managed service.

## Objective

- Configure managed backups with Recovery Service on OCI.
- Enable low- or zero-data-loss protection when real-time data protection is available.
- Create or select a protection policy.
- Configure networking, IAM, subnet, and automatic backups.
- Validate backup, restore, and operations.

## Services involved

- Oracle Cloud Infrastructure IAM
- Oracle Cloud Infrastructure Networking
- Oracle Database Autonomous Recovery Service
- Oracle Base Database Service or Exadata Database Service
- OCI Cloud Guard/Monitoring, when applicable

## Supported scenarios

Use this guide for:

- Oracle Base Database Service on OCI.
- Oracle Exadata Database Service on Dedicated Infrastructure.
- Oracle Exadata Database Service on Exascale Infrastructure, when supported in the region and version.
- Oracle Database@Azure, Oracle Database@Google Cloud, or Oracle Database@AWS, when the multicloud subscription is enabled.

For on-premises databases, the flow is different and uses **Oracle Database Zero Data Loss Cloud Protect**.

## Prerequisites

### Environment information

| Item | Value |
| --- | --- |
| Tenancy | `<tenancy_name>` |
| OCI region | `<region>` |
| Database compartment | `<db_compartment>` |
| Recovery Service compartment | `<recovery_compartment>` |
| VCN | `<vcn_name>` |
| Database subnet | `<db_subnet>` |
| Backup / Recovery Service subnet | `<recovery_service_subnet>` |
| DB System / VM Cluster | `<db_system_or_vm_cluster>` |
| Database/CDB | `<db_name>` |
| DB unique name | `<db_unique_name>` |
| Oracle Database version | `<db_version>` |
| Protection policy | `<protection_policy>` |
| Retention | `<retention_days>` |
| Expected RPO | `<rpo>` |

### Required conditions

- Oracle database supported by Recovery Service.
- Database `COMPATIBLE` set to `19.0.0` or higher.
- Ports open to Recovery Service:
  - `2484`: SQL*Net connection to the RMAN catalog used by Recovery Service.
  - `8005`: backup traffic between the database and Recovery Service.
- Private IPv4 subnet for Recovery Service operations.
- Security rules configured through a Security List or Network Security Group.
- Service limits sufficient for the number of protected databases and backup consumption.
- Manual backups or parallel scripts disabled before enabling automatic Recovery Service backups.

Official references:

- [Overview of Oracle Database Autonomous Recovery Service](https://docs.oracle.com/en-us/iaas/recovery-service/doc/overview-recovery-service.html)
- [Onboarding Oracle Database to Recovery Service](https://docs.oracle.com/en-us/iaas/recovery-service/doc/getting-started-recovery-service.html)
- [Database Autonomous Recovery Service documentation](https://docs.oracle.com/en-us/iaas/recovery-service/index.html)

## High-level architecture

```text
+--------------------------+        Ports 2484 / 8005        +----------------------------------+
| Oracle database on OCI   | --------------------------------> | OCI Autonomous Recovery Service  |
| Base DB / Exadata DB     |                                   | Backup catalog / protected DB    |
+--------------------------+                                   +----------------------------------+
          |
          | VCN / private subnet / security rules
          v
+--------------------------+
| OCI Networking / IAM     |
+--------------------------+
```

## Deployment checklist

- [ ] Confirm database version and compatibility.
- [ ] Confirm the database is in OCI, Exadata DB Service, or a supported multicloud environment.
- [ ] Review Recovery Service limits in the region.
- [ ] Validate IAM policies.
- [ ] Validate VCN, subnet, and security rules.
- [ ] Confirm ports `2484` and `8005`.
- [ ] Create or select a protection policy.
- [ ] Enable automatic backups using Recovery Service.
- [ ] Validate that the protected database was created.
- [ ] Validate the first backup.
- [ ] Validate restore/recover.
- [ ] Configure monitoring and alerts.
- [ ] Record evidence.

## 1. Validate database version

Run on the database:

```sql
SELECT name, db_unique_name, database_role, open_mode
FROM v$database;

SHOW PARAMETER compatible
```

For real-time data protection, validate the minimum release update supported by Oracle documentation for your database version.

## 2. Validate service limits

In the OCI Console:

```text
Governance & Administration
  Tenancy Management
    Limits, Quotas and Usage
      Service: Autonomous Recovery Service
```

Validate:

- Protected database count.
- Space used for the recovery window.
- Regional limits.
- Compartment quotas, if applicable.

## 3. Configure IAM

For Oracle databases in OCI, the Database Service normally already has permissions to access Recovery Service. Still, create administrative permissions for the group operating the service.

Example:

```text
Allow group <recovery_admin_group> to manage recovery-service-family in tenancy
```

More restricted permission for protection policies:

```text
Allow group <recovery_admin_group> to manage recovery-service-policy in compartment <recovery_compartment>
```

Permission for Recovery Service subnets:

```text
Allow group <recovery_admin_group> to manage recovery-service-subnet in compartment <network_compartment>
```

For multicloud, use the official Policy Builder templates for Oracle Database@Azure, Oracle Database@Google Cloud, or Oracle Database@AWS.

## 4. Configure networking

Recovery Service uses a private IPv4 subnet in the same VCN as the database.

For Oracle Database on OCI:

- Exadata Database Service on Dedicated Infrastructure: the backup subnet can be automatically registered as a Recovery Service subnet.
- Base Database Service: the database subnet can be automatically registered as a Recovery Service subnet.
- Optionally, create a dedicated Recovery Service subnet.

Minimum rules:

| Source | Destination | Port | Protocol | Purpose |
| --- | --- | ---: | --- | --- |
| Database subnet | Recovery Service subnet | 2484 | TCP | RMAN catalog / SQL*Net |
| Database subnet | Recovery Service subnet | 8005 | TCP | Backup traffic |

When using an NSG, associate it with the correct resource/subnet for the database type.

## 5. Configure and validate firewall ports 2484 and 8005

> **Owner:** OCI Networking team, with validation by the DBA responsible for the database.

Opening these ports is a required implementation step, not only a prerequisite. Recovery Service uses private connectivity: do not expose either port to `0.0.0.0/0`, the Internet Gateway, or a public subnet.

### 5.1 Identify the protected database network

Record the following values before changing any rule:

| Item | Value |
| --- | --- |
| Database VCN | `<vcn_name>` |
| Database subnet CIDR | `<db_subnet_cidr>` |
| Database VNIC / VM Cluster NSG | `<db_nsg_name_or_ocid>` |
| Recovery Service subnet CIDR | `<recovery_service_subnet_cidr>` |
| Recovery Service subnet NSG, if used | `<recovery_service_nsg_name_or_ocid>` |

For Base Database Service, identify the database VNIC and its NSGs. For Exadata Database Service, identify the client-network VNIC/NSG of each VM in the VM cluster. Do not attach rules to the management network unless that is explicitly your backup network design.

### 5.2 Create the NSG rules

Prefer NSGs over Security Lists so the rule is attached only to the database VNICs. In the OCI Console:

```text
Networking
  Virtual Cloud Networks
    <vcn_name>
      Network Security Groups
        <db_nsg_name>
          Security Rules
            Add Rules
```

Create the following **egress** rules on the NSG associated with the database VNIC(s):

| Direction | Source | Destination | Protocol | Destination port | Stateful | Purpose |
| --- | --- | --- | --- | ---: | --- | --- |
| Egress | Database VNIC / DB NSG | Recovery Service subnet CIDR or Recovery Service NSG | TCP | 2484 | Yes | SQL*Net to Recovery Service catalog |
| Egress | Database VNIC / DB NSG | Recovery Service subnet CIDR or Recovery Service NSG | TCP | 8005 | Yes | Recovery Service backup traffic |

If the Recovery Service subnet is controlled by a customer-managed NSG or Security List, create the corresponding **ingress** rules there:

| Direction | Source | Destination | Protocol | Destination port | Stateful | Purpose |
| --- | --- | --- | --- | ---: | --- | --- |
| Ingress | Database subnet CIDR or DB NSG | Recovery Service subnet / service endpoint | TCP | 2484 | Yes | SQL*Net response path |
| Ingress | Database subnet CIDR or DB NSG | Recovery Service subnet / service endpoint | TCP | 8005 | Yes | Backup traffic response path |

For stateful rules, return traffic is automatically allowed. If your organization uses **stateless** rules, add the matching return rules with source ports `2484` and `8005` on the reverse path.

### 5.3 Alternative: Security List rules

Use this option only when NSGs are not adopted in the VCN. Add the same egress rules to the Security List attached to the database subnet; if the Recovery Service subnet is customer-managed, add the matching ingress rules to its Security List. Because a Security List applies to the entire subnet, confirm the rule scope with the security team before saving.

### 5.4 Validate before enabling automatic backups

1. Confirm that the Recovery Service subnet is private, IPv4-only, and registered in **Database Autonomous Recovery → Recovery Service Subnets**.
2. Confirm that the database VNIC has the intended NSG attached, and that the route tables do not send this private traffic to an Internet Gateway.
3. Enable VCN Flow Logs temporarily, if permitted, to capture rejects during the first backup.
4. After the Recovery Service endpoint has been assigned, test both ports from the database host:

```bash
nc -vz <recovery_service_endpoint> 2484
nc -vz <recovery_service_endpoint> 8005
```

If `nc` is unavailable, ask the network team to validate using Flow Logs and proceed with the first automatic backup. A successful TCP test alone is not sufficient: the final validation is a completed backup and an `Active` protected-database status.

### 5.5 Evidence and rollback

Record the rule OCIDs, NSG/ Security List name, CIDRs, change ticket, test timestamp, and validation result. If the change must be rolled back, remove only the two Recovery Service rules created under the approved change; do not remove pre-existing database network rules.

## 6. Create or select a protection policy

In the OCI Console:

```text
Oracle Database
  Database Autonomous Recovery
    Protection Policies
      Create Protection Policy
```

Define:

- Policy name.
- Compartment.
- Retention.
- Retention lock option, when required for compliance.
- Backup location, when applicable to multicloud.

Custom policies have service-defined retention limits. Validate the permitted range in the current Console and documentation.

## 7. Enable automatic Recovery Service backups

In the OCI Console:

```text
Oracle Database
  Databases
    <database>
      Backups
        Configure automatic backups
```

Select:

- Backup destination: `Autonomous Recovery Service`.
- Protection policy: `<protection_policy>`.
- Retention.
- Real-time data protection, when available and required.

When automatic backups are enabled, Recovery Service may automatically register the Recovery Service subnet, depending on the database type.

## 8. Validate the protected database

In the OCI Console:

```text
Oracle Database
  Database Autonomous Recovery
    Protected Databases
```

Validate:

- The database appears as a protected database.
- Status is `Active` or equivalent.
- Correct protection policy.
- Correct compartment.
- Expected recovery window.
- Most recent backup completed successfully.
- Real-time protection is active, when applicable.

## 9. Validate backup from the database

```sql
SELECT start_time, end_time, status, input_type, output_device_type
FROM v$rman_backup_job_details
ORDER BY start_time DESC
FETCH FIRST 20 ROWS ONLY;
```

Validate archived logs:

```sql
SELECT sequence#, first_time, next_time, applied
FROM v$archived_log
ORDER BY sequence# DESC
FETCH FIRST 20 ROWS ONLY;
```

Validate archive destinations:

```sql
SELECT dest_id, status, error
FROM v$archive_dest
WHERE status <> 'INACTIVE';
```

## 10. Test restore/recover

Start with non-destructive validation:

```rman
RESTORE DATABASE VALIDATE;
RESTORE ARCHIVELOG ALL VALIDATE;
RECOVER DATABASE VALIDATE;
```

For a real test, use an isolated environment, clone, or formally approved restore procedure.

Minimum evidence:

- Active protected database in OCI.
- First backup completed.
- RMAN validate output.
- Restore/recover tested in an isolated environment, when required.
- Alerts configured.

## 11. Monitoring

Monitor:

- Protected database status.
- Most recent backup.
- Recovery window.
- Backup errors.
- Storage consumption.
- Service limits.
- OCI events.
- OCI Monitoring alarms.

Metrics and events should be connected to a notification channel:

```text
Observability & Management
  Monitoring
    Alarm Definitions
      Create Alarm
```

Suggested alarms:

- Backup failure.
- Protected database is not active.
- Consumption approaching a limit.
- No recent backup.
- Redo or real-time protection errors.

## 12. Troubleshooting

### Backup does not start

Validate:

- Automatic backup is enabled.
- Destination is set to Autonomous Recovery Service.
- Protection policy exists in the correct compartment.
- IAM policies allow management.
- Service limits have not been exceeded.

### Network error

Validate:

- Ports `2484` and `8005`.
- Security List or NSG.
- Private IPv4 subnet.
- Routing inside the VCN.
- Whether the subnet was registered as a Recovery Service subnet.

### Protected database does not appear

Validate:

- Compartment selected in the Console.
- Correct region.
- Automatic backup enabled.
- User IAM permissions.
- OCI work request events.

### Real-time data protection is unavailable

Validate:

- Minimum database release update.
- Supported database type.
- Database compatibility.
- Selected protection policy.
- Regional restrictions.

### Parallel manual backups

Before enabling Recovery Service, disable manual scripts or jobs that send operational backups to another destination. Operational backups in two destinations can create data-loss risks or operational inconsistencies.

## 13. Rollback plan

If rollback is required:

- Disable automatic Recovery Service backups.
- Reactivate the previous backup policy only after approval.
- Keep existing backups until a formal retention/cleanup decision.
- Remove alarms or policies only when they are no longer used.
- Record work requests, time, and owner.

## 14. Delivery evidence

| Evidence | Status |
| --- | --- |
| Database compatibility validated | `[ ]` |
| Service limits reviewed | `[ ]` |
| IAM configured | `[ ]` |
| Network ports 2484/8005 allowed | `[ ]` |
| NSG/Security List rule OCIDs recorded | `[ ]` |
| Connectivity tested or Flow Logs reviewed | `[ ]` |
| Protection policy created/selected | `[ ]` |
| Automatic backup enabled | `[ ]` |
| Protected database active | `[ ]` |
| First backup completed | `[ ]` |
| Restore/recover validate executed | `[ ]` |
| Monitoring configured | `[ ]` |

## Appendix A — Change template

```text
Change:
Customer:
Tenancy:
Region:
Compartment:
Database:
DB unique name:
Database type:
Protection policy:
Retention:
Real-time protection:

Start:
End:
Executor:
Validator:

Result:
Open items:
Rollback required:
Notes:
```

## Appendix B — Useful SQL commands

```sql
SELECT name, db_unique_name, database_role, open_mode
FROM v$database;

SHOW PARAMETER compatible

SELECT start_time, end_time, status, input_type, output_device_type
FROM v$rman_backup_job_details
ORDER BY start_time DESC
FETCH FIRST 20 ROWS ONLY;

SELECT dest_id, status, error
FROM v$archive_dest
WHERE status <> 'INACTIVE';
```

---

This repository is independent educational material and does not represent official Oracle documentation. Oracle, Java, and MySQL are trademarks of Oracle Corporation and/or its affiliates. PostgreSQL is a trademark of the PostgreSQL Community Association of Canada.
