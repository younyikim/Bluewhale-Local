# Use case B : Bluewhale AMI로 VOD-SR 자동 시스템 만들기

## Architecture

## Prerequisites

## Workflow

### Step 1. AWS Systems Manager 실행 환경 준비

Bluewhale AMI 기반 EC2 인스턴스에서 FFmpeg을 원격 실행하기 위해서는 먼저 AWS Systems Manager(SSM)에서 해당 인스턴스를 제어할 수 있는 환경을 구성해야 합니다. 본 단계에서는 IAM Role 생성, 권한 설정, 그리고 Bluewhale AMI 기반 EC2 인스턴스 생성까지 VOD-SR 자동화를 위한 기본 기반을 구축합니다.

#### 1. IAM Role 생성 (EC2-SSM-ROLE)

SSM이 EC2 인스턴스를 관리할 수 있도록 전용 IAM Role을 생성합니다.

- **IAM -> Roles -> Create role** 이동
- **Trusted entitiy type** : AWS Service
- **Use case -> EC2** 선택 후 다음 단계 진행
- **Permissions policies**에서 **AmazonSSMManagedInstanceCore** 선택
- Role 이름을 "**EC2-SSM-ROLE**"로 설정하고 생성

<br/>
<img src="../images/vod-sr/step1_iam_01.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_iam_02.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_iam_03.png" width="90%" style="display: block; margin: auto;">
<br/>

#### 2. Inline Policy 추가 (EC2-SSM-POLICY)

SSM Run Commnad 수행에 필요한 추가 권한을 부여합니다.

- **IAM → Roles → EC2-SSM-ROLE** 선택
- **Add permissions → Create inline policy**
- Policy editor에서 **JSON** 탭 선택 후 아래 내용 입력:

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

- Policy 이름을 "**EC2-SSM-POLICY**"로 설정하고 생성

<br/>
<img src="../images/vod-sr/step1_policy_01.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_policy_02.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_policy_03.png" width="90%" style="display: block; margin: auto;">
<br/>

3. Bluewhale AMI 기반 EC2 인스턴스 생성
   생성한 IAM Role을 적용해 SSM 관리가 가능한 Bluewhale AMI 기반 인스턴스를 생성합니다.

- **EC2 → Launch instances**
- **Application and OS Images** 검색창에 _Bluewhale_ 입력
  - “Real-time AI Video Upscaling and Enhancement - Bluewhale (for Coupang)” 선택
- Instance type: **g6e 계열** 선택
- **Advanced details → IAM instance profile → EC2-SSM-ROLE** 선택
- 나머지 설정 완료 후 인스턴스를 생성
- 생성된 인스턴스의 **Instance ID** 확인

<br/>
<img src="../images/vod-sr/step1_ec2_01.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step1_ec2_02.png" width="90%" style="display: block; margin: auto;">
<br/>

### Step 2. AWS Systems Manager Run Command로 FFmpeg 기반 SR 작업 실행하기

이 단계에서는 AWS Systems Manager(SSM)의 Run Command 기능을 활용해 Bluewhale AMI 기반 EC2 인스턴스에서 SR 작업을 원격으로 실행하는 방법을 설명합니다.

#### 1. AWS CLI 설치를 위한 Run Command

Bluewhale AMI 기반 EC2에서 S3 파일을 업로드/다운로드하기 위해 먼저 AWS CLI를 설치합니다.

- **AWS Systems Manager → Run Command → Run a command**
- **Command document**: AWS-RunShellScript 선택
- Command Parameters에 아래 명령 입력:
  ```bash
  sudo apt update
  sudo apt install -y unzip curl
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip awscliv2.zip
  sudo ./aws/install
  aws --version
  ```
- **Target selection → Choose instances manually**에서 Step1에서 생성한 EC2 Instance ID 선택
- **Run** 클릭
- **Targets and outputs**에서 설치 로그를 확인

<br/>
<img src="../images/vod-sr/step2_cli_01.png" width="90%" style="display: block; margin: auto;">
<br/>
<br/>
<img src="../images/vod-sr/step2_cli_02.png" width="90%" style="display: block; margin: auto;">
<br/>

#### 2. FFmpeg 기반 SR 실행을 위한 Run Command

AWS CLI 설치가 완료되면 S3 파일 다운로드 → SR 처리 → 업로드까지 자동 실행되는 스크립트를 Run Command로 실행합니다.

- **AWS Systems Manager → Run Command → Run a command**
- **Command document**: AWS-RunShellScript
- Command Parameters에 아래 중 하나를 입력 :
  - Model A 실행 스크립트
  - Model B 실행 스크립트
- Run Command 실행
- **Targets and outputs → Instance ID** 선택해 Output 확인

#### 2-1. Model A 실행 스크립트

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

#### 2-2. Model B 실행 스크립트

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

#### 2-3. 실행 스크립트 변수 설명

| 변수명                | 설명                            |
| --------------------- | ------------------------------- |
| AWS_ACCESS_KEY_ID     | S3 업로드/다운로드용 Access Key |
| AWS_SECRET_ACCESS_KEY | S3 업로드/다운로드용 Secret Key |
| ORIG_BUCKET           | 원본 파일 S3 버킷               |
| ORIG_FILENAME         | 원본 파일명                     |
| SR_BUCKET SR          | 결과 파일 업로드용 S3 버킷      |
| SR_FILENAME           | 생성된 SR 파일명                |

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
