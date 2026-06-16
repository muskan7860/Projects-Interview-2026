# Database-Related Interview Q&A

## Timeline Reference (keep consistent — don't mix these up)
- **Atos, first ~6 months (2021-2022)**: SQL Server DBA, on-premises, Windows servers, L2 support — **SQL Server 2016**
- **Atos, remaining ~2.2 years (2022-2024)**: Junior DevOps Engineer, AWS banking project — **SQL Server 2019 on RDS**

---

## Q1: What SQL Server version did you work with?
**A (on-prem period):** "On-prem, we were running SQL Server 2016."
**A (AWS/DevOps period):** "On the AWS project, we used SQL Server 2019, hosted on RDS."

## Q2: Was RDS for SQL Server something new/recent when you used it?
**A:** "No — RDS for SQL Server has been available since 2012, so it was a mature, standard offering by the time we used it."

## Q3: What's different about managing SQL Server on RDS vs. on-prem?
**A:** "On RDS, AWS manages the underlying OS, patching, and infrastructure — you don't get OS-level (RDP) access to the database server itself. You manage it through the AWS console and connect to the database using normal tools like SSMS, using the RDS endpoint as the server address. Automated backups and snapshots are built-in and configurable, and high availability (Multi-AZ) is essentially a setting rather than something you build and maintain manually, which is very different from on-prem where we handled backups, patching, and failover setup ourselves."

## Q4: How did your DBA background help in the DevOps role?
**A:** "It meant I could do faster first-level triage on anything database-related — connection issues, timeouts, performance symptoms — and have more informed conversations with the dedicated DBA team when something needed deeper investigation, rather than treating the database as a black box like a typical infra-only engineer might."

## Q5: What DB-related troubleshooting did you personally handle in the AWS project?
**A:** "Mostly infrastructure-level and first-response triage:
- Connection timeouts — checking RDS health via CloudWatch, security group rules, app-side connection pool settings
- Performance symptom triage — checking CloudWatch metrics (CPU, IOPS, connections, latency) as the first step before escalating
- Backup/snapshot verification — confirming automated RDS backups were running correctly
- Failover events — verifying the application reconnected properly after a Multi-AZ failover, and checking RDS events for the root cause
For deeper issues like query optimization, execution plan tuning, or index design, I'd escalate to our dedicated DBA team with my initial findings."

## Q6: What's the difference between Multi-AZ and a Read Replica in RDS?
**A:** "Multi-AZ is for high availability — it maintains a synchronous standby copy in a different Availability Zone, used only for failover, not for serving regular read traffic. A Read Replica is for scaling read performance — it's an asynchronous copy that can actively serve read-only queries to offload traffic from the primary, but it's not automatically used for failover in the same way." *(Note: know this distinction, even if you didn't personally configure read replicas — it's a very common follow-up question.)*

## Q7: How do backups work in RDS vs. on-prem SQL Server?
**A (RDS):** "RDS handles automated backups for you — you configure a retention period (e.g., 7 days), and it takes daily snapshots plus continuous transaction log backups, allowing point-in-time restore within that window."
**A (on-prem, from your real DBA experience):** "On-prem, we managed backup jobs ourselves — full, differential, and transaction log backups on a schedule, verified as part of our go-live checklist process, which I actually built in PowerShell to automate that verification."

## Q8: What is TempDB, and what issues can occur with it? (you mentioned this from real experience — own it)
**A:** "TempDB is a system database SQL Server uses for temporary objects — temp tables, sorting operations, version stores for things like snapshot isolation. Common issues include TempDB filling up disk space due to a long-running query creating large temp objects, or contention/performance issues when many sessions are heavily using TempDB simultaneously. On-prem, I worked alongside senior DBAs on troubleshooting TempDB-related issues as part of L2 support."

## Q9: What is database migration, and what was your role in it?
**A:** "We performed table and database migrations using existing scripts the team maintained — defining the migration steps, running them, and verifying data integrity after migration. I executed these using established scripts and procedures rather than designing migration strategy from scratch, which senior DBAs typically owned."

## Q10: What was the PowerShell go-live checklist you built — walk me through it?
**A:** "Before a SQL Server went live or before a major change, we needed to verify a set of things were correctly configured — database backups were running, compatibility level was set correctly for the SQL Server version, last reboot time, disk space, and a few other server health indicators. I automated this into a PowerShell script that checked all of these and produced a checklist report, instead of someone manually verifying each item — it reduced manual effort and the client specifically appreciated this."

## Q11: What's "compatibility level" in SQL Server, and why does it matter?
**A:** "It controls which version's query optimizer behavior the database uses, even if the SQL Server engine itself has been upgraded. Sometimes after an upgrade, the compatibility level is left at an older setting intentionally or by mistake, which can affect query performance or available features — that's why it was part of our go-live checklist, to make sure it was set correctly for the environment."

## Q12: Active sessions, locks, blocking — how would you check for these? (general knowledge, even if "senior was troubleshooting" in your real experience)
**A:** "You can check active sessions and what they're doing using `sp_who2` or querying `sys.dm_exec_sessions` and `sys.dm_exec_requests`. For blocking specifically, you'd look at the `blocking_session_id` column to identify which session is blocking others, then investigate what that session is running and how long it's held the lock."
