---
title: terraform 的安装
date: 2023-05-08 22:12:31
categories:   
- 技术文档  
tags:   
- terraform 
---

## 安装
1、下载适用于您的操作系统的程序包：[点击官网](https://developer.hashicorp.com/terraform/downloads)

2、 解压复制到对于的bin目录

3、 执行查看
```bash
$ terraform version
Usage: terraform [global options] <subcommand> [args]
Terraform v1.4.4
on linux_amd64
```

## 配置
保存 providers 到统一的位置， 这样就不用每次都下载，都会做软连接
```bash
$ cat << EOF | tee > ~/.terraformrc
plugin_cache_dir   = "$HOME/.terraform.d/plugin-cache"
disable_checkpoint = true
EOF
```

## 命令
```bash
# 在一个空目录创建一个main的tf文件， 同时配置好 providers
$ sudo touch main.tf
# 初始化 providers，下载sdk包，保存到当前目录的 .terraform 目录里
$ sudo terraform init
# 与本地的状态对比一下是删除还是创建资源，仅查看
$ sudo terraform plan
# 应用main.tf文件到环境中 ，不用确认加上 --auto-approve
$ sudo terraform apply
# 删除main.tf指定的所有资源， 不用确认加上 --auto-approve
$ sudo terraform destroy
# 已有的环境中导入资源
$ sudo terraform import
# 查看所有帮助
$ sudo terraform --help
```

## 配置main.tf
```text
terraform {
  required_providers {
    kubernetes = {
      source = "hashicorp/kubernetes"
      version = "2.20.0"
    }
  }
}

provider "kubernetes" {
  # Configuration options
}
```

找对应的providers的官网文档

kubernets：[https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/guides/getting-started](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/guides/getting-started)

alicloud的ACK: [https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/cs_kubernetes](https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/cs_kubernetes)