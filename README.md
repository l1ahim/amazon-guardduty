# Deep Security Amazon GuardDuty Integration

An AWS Lambda function to create a joint workflow between Amazon GuardDuty and Deep Security.

Here's the general workflow of each of these services:

![General Workflow](https://github.com/deep-security/amazon-guardduty/blob/master/docs/general-workflow.png?raw=true "General Trend Micro Deep Security and Amazon GuardDuty workflows")

Combined together, Deep Security can use the intelligence and insight generated by Amazon GuardDuty to make better decisions about the security of your Amazon EC2 instances and Amazon ECS hosts.

![Integrated Workflow](https://github.com/deep-security/amazon-guardduty/blob/master/docs/integrated-workflow.png?raw=true "Integrated Trend Micro Deep Security and Amazon GuardDuty workflow")

## Workflows

This integration supports workflows around the following findings:

- **Recon:EC2/PortProbeUnprotectedPort** when this finding is raised:
	- Deep Security will run a recommendation scan to ensure the instance has an appropriate security policy
	- If requested, Deep Security will also enable intrusion prevention on the instance if it isn't already active

- **Recon:EC2/Portscan** when this finding is raised:
	- Deep Security will run a recommendation scan to ensure the instance has an appropriate security policy
	- If requested, Deep Security will also enable intrusion prevention on the instance if it isn't already active

- **UnauthorizedAccess:EC2/MaliciousIPCaller.Custom** when this finding is raised:
	- Deep Security will run a recommendation scan to ensure the instance has an appropriate security policy
	- Deep Security will run an integrity scan to ensure that critical files have not changed
	- Deep Security will run a malware scan to look for infections or malicious activity
	- If requested, Deep Security will also enable intrusion prevention on the instance if it isn't already active

- **UnauthorizedAccess:EC2/SSHBruteForce** or **UnauthorizedAccess:EC2/RDPBruteForce** when this finding is raised:
	- Deep Security will run a recommendation scan to ensure the instance has an appropriate security policy
	- Deep Security will run an integrity scan to ensure that critical files have not changed
	- If requested, Deep Security will also enable intrusion prevention and integrity monitoring on the instance if it isn't already active

- **CryptoCurrency:\*** when any of these findings are raised:
	- Deep Security will run a malware scan to ensure the instance hasn't been infected
	- If requested, Deep Security will also enable anti-malware on the instance if it isn't already active

- **Backdoor:EC2\*** when any of these findings are raised:
	- Deep Security will run a malware scan to ensure the instance hasn't been infected
	- Deep Security will run an integrity scan to ensure that critical files have not changed
	- If requested, Deep Security will also enable anti-malware on the instance if it isn't already active
	- Deep Security will not enable integrity monitoring because the instance may already be compromised, it will simply run a scan if integrity monitoring is already enable to detect any changes to critical files

- **Trojan:EC2\*** when any of these findings are raised:
	- Deep Security will run a malware scan to ensure the instance hasn't been infected
	- Deep Security will run an integrity scan to ensure that critical files have not changed
	- If requested, Deep Security will also enable anti-malware on the instance if it isn't already active
	- Deep Security will not enable integrity monitoring because the instance may already be compromised, it will simply run a scan if integrity monitoring is already enable to detect any changes to critical files
 
## Support

This is a community project and while you will see contributions from the Deep Security team, there is no official Trend Micro support for this project. The official documentation for the Deep Security APIs is available from the [Trend Micro Online Help Centre](http://docs.trendmicro.com/en-us/enterprise/deep-security.aspx). 

Tutorials, feature-specific help, and other information about Deep Security is available from the [Deep Security Help Center](https://help.deepsecurity.trendmicro.com/Welcome.html). 

For Deep Security specific issues, please use the regular Trend Micro support channels. For issues with the code in this repository, please [open an issue here on GitHub](https://github.com/deep-security/amazon-guarduty/issues).

## Policies In Deep Security

Deep Security uses policies to apply security controls and configurations to your EC2 instances and ECS hosts. These policies allow fine-grained control over the protection that Deep Security applies. **But** in most cases, you don't need to dive into the specifics. The anti-malware and web reputation controls are basically "set and forget". The firewall has limited use within AWS as security groups do a fantastic job (though you still may want a unified firewall across all environments or to leverage it's advanced logging capabilities). 

The intrusion prevention, integrity monitoring, and log inspection controls allow for the automatic application of rules based on recommendations made by Deep Security. It is [**strongly recommended**](https://help.deepsecurity.trendmicro.com/best-practice-guide.html) that you implement this feature. For your Base Policy (or any policy), simply check the option for each of the modules on the detail page for that module.

![Automatically Implement Rule Recommendations](https://github.com/deep-security/amazon-guardduty/blob/master/docs/ds-automatically-apply-rules.png?raw=true "Automatically implement rule recommendations (when possible)")

For the record, [Application Control is also very simple to use](https://help.deepsecurity.trendmicro.com/Protection-Modules/Application-Control/detect-drift.html?Highlight=application%20control) but should be built into your CI/CD workflow. 


## Permissions In Deep Security

Deep Security has a strong role-based access control (RBAC) system built in. In order for these AWS Lambda functions to query Deep Security, they require credentials to sign in.

Here's the recommend configuration in order to implement this with the least amount of privileges possible within Deep Security.

### Role

1. Create a new Role with a unique, meaningful name  (Administration > User Manager > Roles > New...)
1. Under "Access Type", check "Allow Access to web services API"
1. Under "Access Type", **uncheck** "Allow Access to Deep Security Manager User Interface"
1. On the "Computer Rights" tab, select either "All Computers" or "Selected Computers:" ensuring that only the greyed out "View" right (under "Allow Users to:") is selected
1. On the "Policy Rights" tab, select "Selected Policies" and ensure that no policies are selected (this makes sure the role grants no rights to user for any policies)
1. On the "User Rights" tab, ensure that "Change own password and contact information only" is selected
1. On the "Other Rights" tab, ensure that the default options remain with only "View-Only" and "Hide" assigned as permissions

### User

1. Create a new User with a unique, meaningful name (Administration > User Manager > Users > New...)
1. Set a unique, complex password
1. Fill in other details as desired
1. Set the Role to the role you created in the previous section.

Make sure you assign the Role to the user. This will ensure that your API access has the minimal permissions possible, which reduces the risk if the credentials are exposed.

## AWS Lambda Configuration

Please make sure to configure the following;

- Handler: RespondToAmazonGuardDutyViaDeepSecurity.lambda_handler
- Role: basic Lambda invoke is required. Consider adding [AWS X-Ray permissions](http://docs.aws.amazon.com/xray/latest/devguide/xray-permissions.html) for better visibility
- (Advanced Settings) Memory: 128 MB
- (Advanced Settings) Timeout: 5m 0s

### AWS Lambda Function Environment Variables

AWS Lambda now encrypts environment variables by default using a service level key. If you want to, you can override this choice and use a speciifc key managed by KMS. The combination of encrypted by default and a strong RBAC strategy in Deep Security should be sufficient to address most threat models for most risk tolerance levels. If you want to use a specific key under KMS, please follow [the instructions](http://docs.aws.amazon.com/lambda/latest/dg/tutorial-env_console.html) in the AWS Lambda documentation.

<table>
<tr>
  <th>Environment Variable</th>
  <th>Expected Value Type</th>
  <th>Description</th>
</tr>
<tr>
  <td>dsUsername</td>
  <td>string</td>
  <td>The username of the Deep Security account to use for querying anti-malware status</td>
</tr>
<tr>
  <td>dsPassword</td>
  <td>string</td>
  <td>The password for the Deep Security account to use for querying anti-malware status. This password is readable by any identity that can access the AWS Lambda function. Use only the bare minimum permissions within Deep Security (see note below)</td>
</tr>
<tr>
  <td>dsTenant</td>
  <td>string</td>
  <td><i>Optional as long as dsHostname is specified</i>. Indicates which tenant to sign in to within Deep Security</td>
</tr>
<tr>
  <td>dsHostname</td>
  <td>string</td>
  <td><i>Optional as long as dsTenant is specified</i>. Defaults to Deep Security as a Service. Indicates which Deep Security manager the rule should sign in to</td>
</tr>
<tr>
  <td>dsPort</td>
  <td>int</td>
  <td><i>Optional</i>. Defaults to 443. Indicates the port to connect to the Deep Security manager on</td>
</tr>
<tr>
  <td>dsIgnoreSslValidation</td>
  <td>int (0 or 1)</td>
  <td><i>Optional</i>. 0 for false, 1 for true. Use only when connecting to a Deep Security manager that is using a self-signed SSL certificate</td>
</tr>
<tr>
	<td>slackURL</td>
	<td>string</td>
	<td><i>Optional</i>. The incoming webhook URL for the Slack team and channel you would like to send updates and findings to. These messages include contextual information as well as details on the actions taken as a result of the finding</td>
</tr>
<tr>
	<td>enableModules</td>
	<td>int (0 or 1)</td>
  <td><i>Optional</i>. 0 for false, 1 for true. When true, the function will enable the required modules for specific actions (e.g., enable anti-malware in cases where a malware scan is warranted)</td>
</tr>
</table>

During execution, this function will sign in to the Deep Security API. You should setup a dedicated API access account to do this. Deep Security contains a robust role-based access control (RBAC) framework which you can use to ensure that this set of credentials has the least amount of privileges to success. This function requires view and change access to one or more computers within Deep Security.

### SSL Certificate Validation

If the Deep Security Manager (DSM) you're connecting to was installed via software of the AWS Marketplace, there's a chance that it is still using the default, self-signed SSL certificate. By default, python checks the certificate for validity which it cannot do with self-signed certificates.

If you are using self-signed certificates, please use the ```dsIgnoreSslValidation``` environment variable.

When you use this variable, you're telling python to ignore any certificate warnings. These warnings should be due to the self-signed certificate but *could* be for other reasons. It is strongly recommended that you have alternative mitigations in place to secure your DSM. 

When the flag is set, you'll see this warning block in the logs;

```bash
***********************************************************************
* IGNORING SSL CERTIFICATE VALIDATION
* ===================================
* You have requested to ignore SSL certificate validation. This is a less secure method 
* of connecting to a Deep Security Manager (DSM). Please ensure that you have other 
* mitigations and security controls in place (like restricting IP space that can access 
* the DSM, implementing least privilege for the Deep Security user/role accessing the 
* API, etc).
*
* During script execution, you'll see a number of "InsecureRequestWarning" messages. 
* These are to be expected when operating without validation. 
***********************************************************************
```

And during execution you may see lines similar to;

```python
.../requests/packages/urllib3/connectionpool.py:789: InsecureRequestWarning: Unverified HTTPS request is being made. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.org/en/latest/security.html
```

These are expected warnings. Can you tell that we (and the python core teams) are trying to tell you something? If you're interesting in using a valid SSL certificate, you can get one for free from [Let's Encrypt](https://letsencrypt.org), [AWS themselves](https://aws.amazon.com/certificate-manager/) (if your DSM is behind an ELB), or explore commercial options (like the [one from Trend Micro](http://www.trendmicro.com/us/enterprise/cloud-solutions/deep-security/ssl-certificates/)).

## Testing

The integration is triggered when Amazon GuardDuty generates a finding. It can be useful to force the generation of a finding. The simplest of these is a portscan. Run the following ```nmap``` command to trigger a finding.

```nnamp -sX DESTINATION_IP```

A faster feedback loop is to use the set of sample events included in the ```sample-events``` folder in this repo. Simply add these to AWS Lambda was test events and run them against the integration function. This will test the workflow on demand.