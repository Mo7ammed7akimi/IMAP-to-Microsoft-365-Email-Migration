# IMAP to Microsoft 365 Email Migration

Documentation and supporting files for migrating mailboxes from a generic IMAP server to Microsoft 365 (Exchange Online).

## Contents

- [Overview](#overview)
- [Scope and limitations](#scope-and-limitations)
- [Prerequisites](#prerequisites)
- [Repository structure](#repository-structure)
- [Migration plan](#migration-plan)
- [Method 1: Exchange admin centre](#method-1-exchange-admin-centre)
- [Method 2: PowerShell](#method-2-powershell)
- [Method 3: imapsync](#method-3-imapsync)
- [DNS cutover](#dns-cutover)
- [Post-migration tasks](#post-migration-tasks)
- [Troubleshooting](#troubleshooting)
- [Rollback](#rollback)

## Overview

This project describes how to move email from an existing IMAP provider to Microsoft 365. The migration copies mailbox content to Exchange Online while the source server stays active. Mail flow is switched to Microsoft 365 only after the data has been synchronised and verified.

Typical flow:

1. Prepare the Microsoft 365 tenant, domain and mailboxes.
2. Create a migration endpoint and batch.
3. Run an initial sync and, if needed, incremental syncs.
4. Update DNS (MX, Autodiscover, SPF, DKIM, DMARC).
5. Run a final sync, then complete the batch.
6. Reconfigure clients and decommission the old server.

## Scope and limitations

IMAP migration copies **email only**. The following are not migrated and must be handled separately:

- Contacts
- Calendars
- Tasks

Other limitations of the built-in IMAP migration:

- Only items in the Inbox and other mail folders are migrated.
- A maximum of 500,000 items per mailbox is migrated, starting with the newest.
- The maximum size of a single email is 35 MB.
- Target mailboxes must exist and be licensed before the batch starts.
- Passwords (or an admin-level credential) for every source mailbox are required.

Check Microsoft's current documentation, as limits can change.

## Prerequisites

| Requirement | Details |
|---|---|
| Microsoft 365 tenant | Global Administrator or Exchange Administrator role |
| Custom domain | Added and verified in the Microsoft 365 admin centre |
| Licences | One Exchange Online licence per mailbox, assigned |
| Source server | Hostname, port (normally 993 with TLS), and IMAP enabled |
| Source credentials | Username and password per mailbox, or an admin account with delegated access |
| Network | Source IMAP server reachable from the internet |
| DNS access | Ability to edit the domain's DNS records |

Before starting, confirm that:

- [ ] All user accounts exist in Microsoft 365 and are licensed.
- [ ] The source IMAP server accepts connections from Exchange Online.
- [ ] A mailbox inventory (addresses and sizes) has been recorded.
- [ ] The DNS TTL for MX records has been lowered (for example to 300 seconds) at least 48 hours before cutover.
- [ ] Users have been informed of the migration window.

## Repository structure

```
.
├── README.md
├── migration-users.csv.example   # Template for the migration batch CSV
├── scripts/
│   ├── create-endpoint.ps1       # Creates the IMAP migration endpoint
│   ├── create-batch.ps1          # Creates and starts the migration batch
│   └── check-status.ps1          # Reports migration progress
└── docs/
    └── cutover-checklist.md      # Cutover day checklist
```

Adjust this section to match the files actually included in the repository.

## Migration plan

| Phase | Task | Owner |
|---|---|---|
| 1 | Audit source mailboxes and sizes | Administrator |
| 2 | Create users and assign licences | Administrator |
| 3 | Lower DNS TTL | Administrator |
| 4 | Create migration endpoint and batch | Administrator |
| 5 | Initial synchronisation | Automatic |
| 6 | Validate migrated data | Administrator / users |
| 7 | Update MX and related DNS records | Administrator |
| 8 | Final synchronisation and batch completion | Administrator |
| 9 | Reconfigure clients | Users / helpdesk |
| 10 | Decommission source server after retention period | Administrator |

## Method 1: Exchange admin centre

### 1. Prepare the CSV file

Create `migration-users.csv` with the following columns:

```csv
EmailAddress,UserName,Password
user1@example.com,user1@example.com,SourcePassword1
user2@example.com,user2@example.com,SourcePassword2
```

- `EmailAddress`: the target Microsoft 365 mailbox.
- `UserName`: the login name on the source IMAP server.
- `Password`: the password on the source IMAP server.

> **Security note:** The CSV contains plain-text passwords. Do not commit it to version control. Add it to `.gitignore` and delete it once the migration is complete.

### 2. Create the migration batch

1. Open the [Exchange admin centre](https://admin.exchange.microsoft.com).
2. Go to **Migration** and select **Add migration batch**.
3. Choose **Migrate to Exchange Online**, then **IMAP migration**.
4. Create a migration endpoint:
   - IMAP server: `imap.example.com`
   - Authentication: Basic
   - Encryption: SSL
   - Port: 993
5. Upload the CSV file.
6. Set the batch name and notification recipients.
7. Choose to start the batch automatically or manually.

### 3. Monitor progress

In **Migration**, open the batch to view per-mailbox status, items synced and any errors. Download the report for failed items.

## Method 2: PowerShell

Connect to Exchange Online:

```powershell
Install-Module ExchangeOnlineManagement -Scope CurrentUser
Connect-ExchangeOnline -UserPrincipalName admin@example.com
```

Create the migration endpoint:

```powershell
New-MigrationEndpoint -IMAP `
    -Name "IMAP-Source" `
    -RemoteServer "imap.example.com" `
    -Port 993 `
    -Security Ssl
```

Test the endpoint:

```powershell
Test-MigrationServerAvailability -IMAP `
    -RemoteServer "imap.example.com" `
    -Port 993 `
    -Security Ssl
```

Create and start the batch:

```powershell
$csv = [System.IO.File]::ReadAllBytes("C:\migration\migration-users.csv")

New-MigrationBatch -Name "IMAP-Batch-01" `
    -SourceEndpoint "IMAP-Source" `
    -CSVData $csv `
    -AutoStart
```

Check status:

```powershell
Get-MigrationBatch -Identity "IMAP-Batch-01" | Format-List Status, TotalCount, SyncedCount, FailedCount
Get-MigrationUser -BatchId "IMAP-Batch-01" | Get-MigrationUserStatistics
```

Complete or remove the batch after cutover:

```powershell
Complete-MigrationBatch -Identity "IMAP-Batch-01"
Remove-MigrationBatch -Identity "IMAP-Batch-01"
```

## Method 3: imapsync

[imapsync](https://imapsync.lamiral.info/) is an alternative when the built-in migration is not suitable, for example when more control over folders or filtering is required.

```bash
imapsync \
  --host1 imap.example.com --user1 user@example.com --password1 'SourcePassword' --ssl1 \
  --host2 outlook.office365.com --user2 user@example.com --password2 'TargetPassword' --ssl2 \
  --automap
```

Notes:

- Run with `--dry` first to preview the actions.
- Microsoft 365 may require an app password or modern authentication, depending on the tenant's security settings. Check the tenant's IMAP and authentication configuration.
- imapsync can be re-run safely to copy only new messages.

## DNS cutover

Perform these changes once the initial sync is complete and verified.

| Record | Type | Value (example) | Purpose |
|---|---|---|---|
| MX | MX | `example-com.mail.protection.outlook.com` (priority 0) | Routes incoming mail to Microsoft 365 |
| Autodiscover | CNAME | `autodiscover.outlook.com` | Client auto-configuration |
| SPF | TXT | `v=spf1 include:spf.protection.outlook.com -all` | Authorises Microsoft 365 to send mail |
| DKIM | CNAME | Values provided in the Microsoft Defender portal | Message signing |
| DMARC | TXT | `v=DMARC1; p=none; rua=mailto:dmarc@example.com` | Reporting and policy |

The exact MX value is shown in the Microsoft 365 admin centre under **Settings > Domains**. Remove any old MX records after the change and merge SPF entries rather than publishing more than one SPF record.

After the MX change, allow time for propagation, then run a final sync to capture any mail that arrived on the source server during the switch.

## Post-migration tasks

- [ ] Run a final incremental sync and complete the batch.
- [ ] Verify mailbox item counts against the inventory.
- [ ] Send and receive test messages (internal, external, large attachments).
- [ ] Confirm SPF, DKIM and DMARC pass using a message header analyser.
- [ ] Set up Outlook profiles or mobile clients for each user.
- [ ] Migrate contacts and calendars separately (export/import as PST or CSV).
- [ ] Reset the DNS TTL to its normal value.
- [ ] Keep the source server running read-only for an agreed retention period (for example 30 days).
- [ ] Delete `migration-users.csv` and any stored credentials.

## Troubleshooting

| Problem | Likely cause | Action |
|---|---|---|
| Endpoint test fails | Firewall, wrong port or TLS setting | Check that port 993 is open and the certificate is valid |
| Authentication errors per user | Wrong password or IMAP disabled for the account | Verify the credentials by logging in directly to the source server |
| Mailbox not found | Target user not created or not licensed | Create the user and assign a licence, then retry |
| Items skipped | Item over 35 MB or mailbox over 500,000 items | Review the batch report and migrate those items manually |
| Slow migration | Throttling or large mailboxes | Split into smaller batches and run outside peak hours |
| Mail still arriving at old server | MX not yet propagated or high TTL | Check with `nslookup -type=mx example.com` and wait for propagation |
| Duplicate messages | Batch restarted after manual client copy | Avoid copying mail manually while the batch is running |

## Rollback

If problems occur after cutover:

1. Restore the previous MX records.
2. Wait for DNS propagation.
3. Mail sent during the Microsoft 365 period remains in the Exchange Online mailboxes and can be copied back with imapsync or exported if required.

Rollback is only practical while the source server is still active, so avoid decommissioning it too early.

## References

- [Migrate email from an IMAP server to Microsoft 365](https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrating-imap-mailboxes/migrating-imap-mailboxes)
- [Add a domain to Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/admin/setup/add-domain)
- [Set up SPF, DKIM and DMARC](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-about)

## Licence

Add a licence for this repository (for example MIT) or remove this section.
