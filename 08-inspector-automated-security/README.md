AWS: Automate Security Checks with Amazon Inspector
Overview
This project demonstrates how to enable and use Amazon Inspector to perform automated security assessments of AWS resources. Amazon Inspector continuously scans resources for potential security vulnerabilities and provides detailed findings with severity ratings.
In this project, we:
Configure Amazon Inspector to scan a pre-configured Amazon EC2 instance
Analyze vulnerability findings to understand our overall security posture
Create suppression rules to prioritize security risks based on severity ratings
---
Objectives
Activate and explore Amazon Inspector
Review and interpret security findings
Create suppression rules based on finding severity ratings
---
Tasks
Task 1: Activate and Explore Amazon Inspector
1.1 Activate Amazon Inspector
On the AWS Management Console, search for and choose Amazon Inspector.
Choose Get Started.
On the Activate Inspector page, choose Activate this account.
A banner confirms: "Welcome to Inspector. Your first scan is underway."
1.2 Review the Amazon Inspector Summary Dashboard
The dashboard provides a comprehensive overview of security findings across AWS resources. Key sections include:
Environment coverage
Critical findings
Findings with exploit available and fix available
Risk based remediations
> **Note:** Since Amazon Inspector was just enabled, it may take a few minutes for results to appear.
1.3 Enable Hybrid EC2 Scanning
Configure Amazon Inspector to use hybrid scanning mode — combining agent-based and agentless scanning for comprehensive EC2 coverage.
In the left navigation pane, under General settings, choose EC2 scanning settings.
Verify that the scan mode is set to Hybrid.
If not, choose Edit, select Hybrid, and choose Save.
What Hybrid mode means:
Agent-based scanning — for instances managed by AWS Systems Manager (SSM)
Agentless scanning (via EBS snapshots) — for instances not managed by SSM
Navigate to Resources coverage → EC2 instances to view instance scan status.
---
Task 2: Review and Interpret Amazon Inspector Findings
2.1 Examine Vulnerability Findings and Severity Ratings
Navigate to Findings in the left navigation pane. The Severity column shows ratings for all detected vulnerabilities:
Severity	Description
Critical	Immediate action required — highest risk vulnerabilities
High	Significant security risks to be addressed quickly
Medium	Important vulnerabilities to remediate in a timely manner
Low	Minor issues addressable during routine maintenance
Findings detected for the EC2 instance:
Severity	CVE ID	Affected Package
High	CVE-2025-59046	`interactive-git-checkout`
Low	CVE-2025-47278	`flask`
Finding Details Pane includes:
Finding overview — severity, type, and fix availability
Affected packages — vulnerable software and versions
Vulnerability details — technical scoring, CVE identifiers, and classification data
Resource affected — the specific AWS resource impacted
> **What is a CVE?**  
> A CVE (Common Vulnerabilities and Exposures) identifier is a standardized code representing a specific known security vulnerability.  
> Format: `CVE-YEAR-NUMBER` (e.g., `CVE-2025-59046`)  
> Reference: [NVD CVE Detail](https://nvd.nist.gov/vuln/detail/CVE-2025-59046)
2.2 View Findings by Instance and Apply Filters
In the left navigation pane, under Findings, choose By instance.
Select the instance ID link to see findings for that specific EC2 instance.
Use the filter criteria search field → select Severity → High → choose Apply.
---
Task 3: Create a Suppression Rule and Review Updated Findings
3.1 Create a Suppression Rule
Suppression rules automatically filter out lower-priority findings to reduce alert fatigue and keep focus on high-impact risks.
In the left navigation pane, choose Suppression rules.
Choose Create rule.
Fill in the details:
Name: `Suppress low severity findings`
Description: `Suppress low severity findings`
Under Suppression rule filters, choose Add filter → Severity → Low → Apply.
Choose Save.
> **Caution:** Creating suppression rules should be done carefully. Suppressing low-severity findings helps reduce noise, but ensure you are not overlooking potentially important vulnerabilities.
3.2 Review Updated Findings
In the left navigation pane, choose Findings.
Use the Finding status dropdown to toggle between views:
Suppressed — findings hidden by your rule
Show all — all findings including suppressed
Active — only findings not suppressed by your rule
---
Conclusion
By completing this project, we successfully:
✅ Activated and explored Amazon Inspector
✅ Reviewed and interpreted security findings with severity ratings
✅ Created suppression rules to filter low-severity findings and prioritize critical risks
---
Additional Resources
For more information about Amazon Inspector, see Amazon Inspector Documentation.
