---
title: 폐쇄망에서 terraform 사용하기
author: garit
date: 2022-11-26 11:00:00 +0900
categories: [Terraform, Cases]
tags: [terraform, aws]
render_with_liquid: false
---

## 폐쇄망에서 terraform 사용하기

### 상황

폐쇄망에서 Terraform으로 AWS에 인프라를 구성하려고 하니 아래와 같이 terraform 공식 registry에 접속할 수 없어 에러가 발생한다.  
<br/>

![tf-error](https://user-images.githubusercontent.com/67899732/210208324-f18d144f-40e9-40bc-a1f9-cfb2706d31ca.png)

그러므로 필요한 provider 패키지 바이너리 파일을 직접 다운 받아 로컬에 저장해놓고 terraform을 실행할 수 있게 해야 한다.  
<br/>

### AWS Provider 패키지 다운로드하기

terraform에서 쓰는 provider 패키지 바이너리 파일은 아래 URL에서 다운받을 수 있다.  
> https://releases.hashicorp.com/terraform-provider-aws/4.48.0/  
<br/>

- aws 자리에 다른 provider를 넣고 4.48.0 자리에 다른 버전을 넣어서 원하는 파일을 다운로드 할 수 있다.  
<br/>

### terraform init 시, 로컬 경로에서 다운받게 설정하기

**1.위에서 다운받은 zip 파일을 terraform을 실행하는 서버로 옮긴다.**

- 압축을 풀고 아래 경로로 복사한다.

> unzip terraform-provider-aws_4.48.0_linux_amd64.zip
> # provider 패키지를 다운받을 경로
> mkdir -p /usr/share/terraform/providers/registry.terraform.io/hashicorp/aws/4.48.0/linux_amd64/
> cp ./terraform-provider-aws_v4.48.0_x5 /usr/share/terraform/providers/registry.terraform.io/hashicorp/aws/4.48.0/linux_amd64/
> # plugin cache 경로
> mkdir -p ~/.terraform.d/plugins/registry.terraform.io/hashicorp/aws/4.48.0/linux_amd64/
> cp ./terraform-provider-aws_v4.48.0_x5 ~/.terraform.d/plugins/registry.terraform.io/hashicorp/aws/4.48.0/linux_amd64/

<br/>

**2.terraformrc 파일을 생성하여 공식저장소 대신 다운받을 경로를 지정한다.**

> $ vi ~/.terraformrc

```bash
# plugin cache 저장 경로
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
disable_checkpoint = true
provider_installation {
	filesystem_mirror {
	    # 공식저장소 대신 init시 패키지를 다운받을 경로
		path = "/usr/share/terraform/providers"
		include = ["registry.terraform.io/hashicorp/*",.....]
	}
	direct {
		exclude = ["registry.terraform.io/hashicorp/*",.....]
	}
}

```

### terraform init 테스트

vi ~/terraform-test/aws-ec2/main.tf

```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 3.0.0"
    }
  }

  required_version = ">= 1.2.0"
}

provider "aws" {
  region  = "us-west-2"
}

resource "aws_instance" "app_server" {
  ami           = "ami-830c94e3"
  instance_type = "t2.micro"

  tags = {
    Name = "ExampleAppServerInstance"
  }
}

```

> $ terraform init 

![tf-init](https://user-images.githubusercontent.com/67899732/210210939-fcf0e921-11ef-4255-996f-3cbedfce77d3.png)


에러가 발생하지 않고 init에 성공했다.  
<br/>

그럼 이제 AWS VPC부터 생성해보자.  
<br/>


참고
- [https://developer.hashicorp.com/terraform/tutorials/aws-get-started](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)
- [https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-build#troubleshooting](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-build#troubleshooting)
- [https://releases.hashicorp.com/terraform-provider-aws/4.48.0/](https://releases.hashicorp.com/terraform-provider-aws/4.48.0/)


