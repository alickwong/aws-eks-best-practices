!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 容器镜像扫描

镜像扫描是一项自动化漏洞评估功能,通过扫描容器镜像中广泛的操作系统漏洞,帮助提高应用程序的安全性。

目前,Amazon Elastic Container Registry (ECR) 只能扫描 Linux 容器镜像中的漏洞。但是,也有一些第三方工具可以集成到现有的 CI/CD 管道中,用于扫描 Windows 容器镜像。

* [Anchore](https://anchore.com/blog/scanning-windows-container-images/)
* [PaloAlto Prisma Cloud](https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-admin-compute/vulnerability_management/windows_image_scanning.html)
* [Trend Micro - Deep Security Smart Check](https://www.trendmicro.com/en_us/business/products/hybrid-cloud/smart-check-image-scanning.html)

要了解如何将这些解决方案与 Amazon Elastic Container Repository (ECR) 集成,请查看:

* [Anchore, 在 Amazon Elastic Container Registry (ECR) 上扫描镜像](https://anchore.com/blog/scanning-images-on-amazon-elastic-container-registry/)
* [PaloAlto, 在 Amazon Elastic Container Registry (ECR) 上扫描镜像](https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-admin-compute/vulnerability_management/registry_scanning0/scan_ecr.html)
* [TrendMicro, 在 Amazon Elastic Container Registry (ECR) 上扫描镜像](https://cloudone.trendmicro.com/docs/container-security/sc-about/)
