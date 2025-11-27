# Use case B : Building an Automated VOD-SR System with the Bluewhale AMI

## Architecture

## Prerequisites

## Workflow

### Step 1. Preparing the AWS Systems Manager Execution Environment

To remotely run FFmpeg on an EC2 instance based on the Bluewhale AMI, you must first configure an environment that allows AWS Systems Manager (SSM) to manage that instance. In this step, you will create the required IAM role, set the necessary permissions, and launch an EC2 instance using the Bluewhale AMI to establish the foundational infrastructure for automated VOD-SR processing.

#### 1. Creating an IAM Role (EC2-SSM-ROLE)

Create a dedicated IAM role that allows AWS Systems Manager (SSM) to manage your EC2 instance.

- Navigate to **IAM -> Roles -> Create role**
- **Trusted entitiy type** : AWS Service
- Select **Use case -> EC2**, then proceed to the next step
- Under **Permissions policies**, select **AmazonSSMManagedInstanceCore**
- Set the role name to "**EC2-SSM-ROLE**" and create the role

<br/>
<img src="../images/vod-sr/step1_iam_01.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_iam_02.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_iam_03.png" width="90%" style="display: block; margin: auto;">
<br/>

#### 2. Adding an Inline Policy (EC2-SSM-POLICY)

Add additional permissions required for executing SSM Run Command.

- Go to **IAM → Roles → EC2-SSM-ROLE**
- Select **Add permissions → Create inline policy**
- In Policy editor, switch to the **JSON** tab and enter the following content:

  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ssm:SendCommand",
          "ssm:ListCommands",
          "ssm:ListCommandInvocations"
        ],
        "Resource": "*"
      }
    ]
  }
  ```

- Set Policy name to "**EC2-SSM-POLICY**" and create the policy

<br/>
<img src="../images/vod-sr/step1_policy_01.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_policy_02.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_policy_03.png" width="90%" style="display: block; margin: auto;">
<br/>

#### 3. Launching an EC2 Instance with the Bluewhale AMI

Launch an EC2 instance based on the Bluewhale AMI and attach the IAM role created in the previous step, enabling the instance to be managed through SSM.

- Go to **EC2 → Launch instances**
- In the **Application and OS Images** search bar, enter _Bluewhale_
  - Select “Real-time AI Video Upscaling and Enhancement - Bluewhale (for Coupang)”
- Choose an instance type from the **g6e** family
- Under **Advanced details → IAM instance profile**, select **EC2-SSM-ROLE**
- Complete the remaining configuration and launch the instance
- After creation, note the **Instance ID** of the EC2 instance

<br/>
<img src="../images/vod-sr/step1_ec2_01.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_ec2_02.png" width="90%" style="display: block; margin: auto;">
<br/>

### Step 2. Executing FFmpeg-Based SR Processing Using AWS Systems Manager Run Command

In this step, you will use the Run Command feature of AWS Systems Manager (SSM) to remotely execute super-resolution (SR) processing on an EC2 instance running the Bluewhale AMI.

#### 1. Running a Command to Install the AWS CLI

To upload or download files from Amazon S3 on the Bluewhale AMI–based EC2 instance, you must first install the AWS CLI.

- Navigate toe **AWS Systems Manager → Run Command → Run a command**
- For **Command document**, select AWS-RunShellScript
- Enter the following commands in the Command Parameters field:
  ```bash
  sudo apt update
  sudo apt install -y unzip curl
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip awscliv2.zip
  sudo ./aws/install
  aws --version
  ```
- Under **Target selection → Choose instances manually**, select the EC2 Instance ID created in Step 1
- Click **Run**
- Review the installation results under **Targets and outputs**

<br/>
<img src="../images/vod-sr/step2_cli_01.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step2_cli_02.png" width="90%" style="display: block; margin: auto;">
<br/>

#### 2. Running a Command to Execute FFmpeg-Based SR Processing

After installing the AWS CLI, you can use Run Command to automatically download the source file from S3, perform the SR (super-resolution) process, and upload the resulting file back to S3.

- Go to **AWS Systems Manager → Run Command → Run a command**
- For **Command document**, select AWS-RunShellScript
- In the Command Parameters field, enter one of the following :
  - Model A execution script
  - Model B execution scropt
- Execute Run Command
- Under **Targets and outputs → Instance ID**, review the command output

#### 2-1. Model A execution script

```bash
### set env-variables
export ONNX_DIR=/usr/local/bluedot/models/
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:/usr/local/bluedot/:/usr/local/TensorRT/lib:$LD_LIBRARY_PATH
export PATH=/usr/local/bluedot:$PATH

export LD_LIBRARY_PATH=/etc/neura-vortex/nvx_packages/plugins/libs:$LD_LIBRARY_PATH
export PYTHONPATH=/etc/neura-vortex/libs/python3.10/site-packages
export VSSCRIPT_PATH=/etc/neura-vortex/nvx_packages/plugins/scripts
export VSPREFIX=/etc/neura-vortex
export PATH=$PATH:/etc/neura-vortex/bin
export VAPOURSYNTH_CONF_PATH=/etc/neura-vortex/vapoursynth.conf

### s3 access key
export AWS_ACCESS_KEY_ID=your_access_key_id
export AWS_SECRET_ACCESS_KEY=your_secret_access_key
export AWS_DEFAULT_REGION=ap-northeast-2

### orig bucket and original filename
ORIG_BUCKET='bluewhale-ami-sample'
ORIG_FILENAME='original_640x480_10s.mp4'

### sr bucket and sr filename
SR_BUCKET='bluewhale-ami-sample-sr'
SR_FILENAME='sr_1280x960_10s.mp4'

### change working directory
cd /root

### original_file : download from s3
aws s3 cp s3://${ORIG_BUCKET}/${ORIG_FILENAME} ./${ORIG_FILENAME}

### ffmpeg : original_file -> SR -> sr_file (Model A)
ffmpeg -y -i ${ORIG_FILENAME} -vf bdnvx_aws,bdsr_coupang_aws,noise=c0s=8:allf=t+u -c:v h264_nvenc -cq 13 -c:a copy ${SR_FILENAME}

### sr_file : upload to s3
aws s3 cp ./${SR_FILENAME} s3://${SR_BUCKET}/${SR_FILENAME}
```

#### 2-2. Model B execution script

```bash
### set env-variables
export ONNX_DIR=/usr/local/bluedot/models/
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:/usr/local/bluedot/:/usr/local/TensorRT/lib:$LD_LIBRARY_PATH
export PATH=/usr/local/bluedot:$PATH

export LD_LIBRARY_PATH=/etc/neura-vortex/nvx_packages/plugins/libs:$LD_LIBRARY_PATH
export PYTHONPATH=/etc/neura-vortex/libs/python3.10/site-packages
export VSSCRIPT_PATH=/etc/neura-vortex/nvx_packages/plugins/scripts
export VSPREFIX=/etc/neura-vortex
export PATH=$PATH:/etc/neura-vortex/bin
export VAPOURSYNTH_CONF_PATH=/etc/neura-vortex/vapoursynth.conf

### s3 access key
export AWS_ACCESS_KEY_ID=your_access_key_id
export AWS_SECRET_ACCESS_KEY=your_secret_access_key
export AWS_DEFAULT_REGION=ap-northeast-2

### orig bucket and original filename
ORIG_BUCKET='bluewhale-ami-sample'
ORIG_FILENAME='original_640x480_10s.mp4'

### sr bucket and sr filename
SR_BUCKET='bluewhale-ami-sample-sr'
SR_FILENAME='sr_1280x960_10s.mp4'

### change working directory
cd /root

### original_file : download from s3
aws s3 cp s3://${ORIG_BUCKET}/${ORIG_FILENAME} ./${ORIG_FILENAME}

### ffmpeg : original_file -> SR -> sr_file (Model B)
ffmpeg -y -i ${ORIG_FILENAME} -vf bdsr_coupang_aws -c:v h264_nvenc -cq 13 -c:a copy ${SR_FILENAME}

### sr_file : upload to s3
aws s3 cp ./${SR_FILENAME} s3://${SR_BUCKET}/${SR_FILENAME}
```

#### 2-3. Description of Script Variables

| Variable NAme         | Description                                             |
| --------------------- | ------------------------------------------------------- |
| AWS_ACCESS_KEY_ID     | Access key used for uploading/downloading files from S3 |
| AWS_SECRET_ACCESS_KEY | Secret key used for uploading/downloading files from S3 |
| ORIG_BUCKET           | S3 bucket containing the source file                    |
| ORIG_FILENAME         | Name of the source file                                 |
| SR_BUCKET SR          | S3 bucket for uploading the SR output file              |
| SR_FILENAME           | Name of the generated SR file                           |

<br/>
<img src="../images/vod-sr/step2_sr_01.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step2_sr_02.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step2_sr_03.png" width="90%" style="display: block; margin: auto;">
<br/>

### Step 3.
