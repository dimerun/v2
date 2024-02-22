# DimeRun v2

DimeRun v2 enables you to provision on-demand self-hosted GitHub Actions runners on AWS EC2. These ephemeral runners are spun up in your own AWS account, providing flexibility and scalability to your CI/CD processes.

## Getting started

### 1. Create a GitHub organization

If you haven't already, sign in to GitHub and create an empty organization.

Click on your profile icon in the top-right corner of the GitHub homepage. Select "Your organizations" from the dropdown menu. Click "New organization".

<img width="200" alt="image" src="https://github.com/dimerun/v2/assets/33595751/d6775140-0e8d-460c-9cfe-cc32b1f48313">

Choose a plan, an organization name, contact email, and organization ownership. Click "Next" to create an organization and finish the setup.

<img width="480" alt="image" src="https://github.com/dimerun/v2/assets/33595751/f172250a-287f-47e3-ada5-ec365065d35c">

### 2. Create a GitHub App

Create a GitHub App that is owned by the organization. For more information, see [Registering a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app). Configure the GitHub App as follows.

<img width="800" alt="image" src="https://github.com/dimerun/v2/assets/33595751/a0abfac2-7e34-48f9-bba3-9986b08f6260">

a. For "Homepage URL", enter https://dime.run/.

b. Under "Permissions," click Repository permissions. Then use the dropdown menus to select the following access permissions.
- Administration: Read and write
- Metadata: Read-only

c. Under "Permissions," click Organization permissions. Then use the dropdown menus to select the following access permissions.
- Self-hosted runners: Read and write

After creating the GitHub App, note the value for "App ID" on the GitHub App's page. You will use this value later.

<img width="800" alt="image" src="https://github.com/dimerun/v2/assets/33595751/5585bc5d-01d2-4bac-96a0-ebe2636d8822">

Under "Private keys", click Generate a private key, and save the `.pem` file. You will use this key later.

<img width="600" alt="image" src="https://github.com/dimerun/v2/assets/33595751/f0c66c92-25ca-4817-a54b-c1cabb75e69d">

In the menu at the top-left corner of the page, click Install app, and next to your organization, click Install to install the app on your organization.

<img width="800" alt="image" src="https://github.com/dimerun/v2/assets/33595751/e98d5a5d-55f7-48bf-9011-ed1b02c0d0f9">

After confirming the installation permissions on your organization, note the app installation ID. You will use it later. You can find the app installation ID on the app installation page, which has the following URL format: https://github.com/organizations/ORGANIZATION/settings/installations/INSTALLATION_ID

<img width="800" alt="image" src="https://github.com/dimerun/v2/assets/33595751/665a43ed-fd97-4ca1-b59f-0d3f3e463286">

### 3. Grant AWS account access

Ephemeral runners are created in your AWS account, which is required to have [AmazonEC2FullAccess permission](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonEC2FullAccess.html). This permission allows the DimeRun v2 service to dynamically provision and manage EC2 instances on your behalf. Ensure that the AWS credentials used by DimeRun have the necessary permissions configured with the appropriate IAM policies.

To generate and list the necessary access key and secret key for IAM users of your AWS account, you can follow the instructions provided in the [AWS documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html).

### 4. Create a JSON config file

You'll need to create a JSON configuration file. This file will contain all the necessary parameters and settings for GitHub and AWS.

Below is a sample configuration file:

```json
{
    "githubConfigURL": "https://github.com/xxx",
    "githubAppAuth": {
        "appId": 000000,
        "appInstallationID": 00000000,
        "appPrivateKeyPath": "./key.pem"
    },
    "runnerGroupId": 1,
    "awsConfig": {
        "accessKey": "xxxxxxxxxxxxxxxxxxxx",
        "secretKey": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "region": "us-east-1",
        "instanceType": "m7a.large"
        "pdVolumeType": "gp2",
        "pdVolumeSize": 10
    },
    "userEmail": "yourname@company.com"
}
```

Modify the configuration as needed according to your specific environment:

- githubConfigURL: Your organization's GitHub URL, which has the following URL format: https://github.com/organizations/ORGANIZATION.
- githubAppAuth: Authentication details for GitHub App.
  - appId: Your GitHub App ID.
  - appInstallationID: Your GitHub App Installation ID.
  - appPrivateKeyPath: Path to the private `.pem` key file for GitHub App authentication.
- runnerGroupId: ID of the runner group. The default is 1, corresponding to the default runner group.
- awsConfig: AWS configuration details.
  - accessKey: Your AWS access key.
  - secretKey: Your AWS secret key.
  - region: AWS region. Currently, our support is limited to `us-xxx`. Please reach out to us if additional regions are required.
  - instanceType: Type of EC2 instances.
  - pdVolumeType: Type of the persistent disk volume.
  - pdVolumeSize: Size of the persistent disk volume.
- userEmail: Your email address. By filling in your email address, you agree to be added to our mailing list.

### 5. Run with configuration

To deploy DimeRun v2, execute the following commands with your configuration file:

```
wget -O dimerunv2 https://github.com/dimerun/v2/releases/download/v2.0.0/dimerun-2.0.0-linux-x86_64
chmod +x ./dimerunv2
./dimerunv2 --config config.json
```

### 6. Trigger in GitHub Actions workflows

When defining workflows in your GitHub repository, specify `dimerunv2` as the execution environment. Update your workflow file as follows:

```diff
- runs-on: ubuntu-22.04
+ runs-on: dimerunv2
```

It will take a while to wait for a runner from DimeRun v2 to come online and pick up your job.

## Using a persistent disk

DimeRun v2 incorporates the functionality to attach a persistent disk to each instance upon execution. The persistent disk serves as durable storage for your application data and configurations.

The default device name of the persistent disk volume on EC2 instances is `/dev/sdb`. However, the device name used in GitHub workflows as a mount point on Linux OS can differ between instance families, such as T2 and T3. T2 uses `/dev/sd[a-z]`, while T3 uses `/dev/nvme[0-26]n1`. Using the `lsblk` command to confirm the device name before mounting is recommended. For more information, see [How to mount Linux volume and keep mount point consistency](https://aws.amazon.com/cn/blogs/compute/how-to-mount-linux-volume-and-keep-mount-point-consistency/). Here are some commonly used EC2 instance types and their corresponding persistent disk mount points:

| Instance Type |	PD Mount Point |
| --- | --- |
| t2.micro |	/dev/xvdb |
| t3.micro |	/dev/xvdb |
| m5.large |	/dev/nvme1n1 |
| m7a.large |	/dev/nvme1n1 |
| c5.xlarge |	/dev/nvme1n1 |
| r5.large |	/dev/nvme1n1 |
| i3.large |	/dev/nvme1n1 |

The following is an example of using persistent disks in GitHub workflows.

```yaml
name: Persistent Disk Demo
on:
  workflow_dispatch:
env:
  PD: /home/runner/.cache
jobs:
  demo:
    runs-on: dimerunv2
    steps:
    - name: Mount Persistent Disk
      run: |
        sudo lsblk
        sudo mkfs.btrfs /dev/nvme1n1 || true
        sudo mkdir -p $PD
        sudo mount /dev/nvme1n1 $PD
        sudo df -hT
        sudo chown -cR runner $PD
    - run: |
        date >> $PD/dates.txt
        cat $PD/dates.txt
        echo "🎉 This job uses persistent disk!"
```

## Deployment

For detailed deployment instructions, please refer to our [wiki](https://github.com/dimerun/v2/wiki).

## Support

If you find a bug or have questions to ask, feel free to discuss it in the [issues](https://github.com/dimerun/v2/issues).
