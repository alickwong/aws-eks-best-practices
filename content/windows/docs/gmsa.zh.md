!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 为 Windows Pod 和容器配置 gMSA

## 什么是 gMSA 帐户

基于 Windows 的应用程序(如 .NET 应用程序)通常使用 Active Directory 作为身份提供者,使用 NTLM 或 Kerberos 协议提供授权/身份验证。

应用程序服务器与 Active Directory 交换 Kerberos 票证需要加入域。Windows 容器不支持加入域,这也不太合理,因为容器是临时资源,会给 Active Directory RID 池带来负担。

但是,管理员可以利用 [gMSA Active Directory](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview) 帐户来协商 Windows 身份验证,用于资源(如 Windows 容器、NLB 和服务器场)。

## Windows 容器和 gMSA 用例

利用 Windows 身份验证的应用程序在以 Windows 容器运行时,可以从 gMSA 中获益,因为 Windows 节点用于代表容器交换 Kerberos 票证。有两种选择可以设置 Windows 工作节点以支持 gMSA 集成:

#### 1 - 加入域的 Windows 工作节点
在此设置中,Windows 工作节点加入 Active Directory 域,并使用 Windows 工作节点的 AD 计算机帐户向 Active Directory 进行身份验证,并检索要与 pod 一起使用的 gMSA 身份。

在加入域的方法中,您可以使用现有的 Active Directory GPO 轻松管理和加固您的 Windows 工作节点;但是,它会产生额外的操作开销和延迟,因为在 Kubernetes 集群终止节点时,Windows 工作节点加入需要额外的重新启动和 Active Directory 垃圾清理。

在以下博客文章中,您将找到有关如何实施加入域的 Windows 工作节点方法的详细分步骤:

[在 Amazon EKS Windows pod 上实现 Windows 身份验证](https://aws.amazon.com/blogs/containers/windows-authentication-on-amazon-eks-windows-pods/)

#### 2 - 无域 Windows 工作节点
在此设置中,Windows 工作节点未加入 Active Directory 域,而是使用"便携式"身份(用户/密码)向 Active Directory 进行身份验证,并检索要与 pod 一起使用的 gMSA 身份。

![](./images/domainless_gmsa.png)

便携式身份是 Active Directory 用户;该身份(用户/密码)存储在 AWS Secrets Manager 或 AWS System Manager Parameter Store 中,并将使用 AWS 开发的插件 ccg_plugin 从 AWS Secrets Manager 或 AWS System Manager Parameter Store 检索此身份,并将其传递给 containerd 以检索 gMSA 身份并使其可用于 pod。
在这种无域的方法中，您可以在使用 gMSA 时受益于在 Windows 工作节点启动期间没有任何 Active Directory 交互,并减少 Active Directory 管理员的操作开销。

在以下博客文章中,您将找到关于如何实施无域 Windows 工作节点方法的详细分步指南:

[Amazon EKS Windows 容器的无域 Windows 身份验证](https://aws.amazon.com/blogs/containers/domainless-windows-authentication-for-amazon-eks-windows-pods/)

#### 重要提示

尽管 pod 能够使用 gMSA 帐户,但也有必要相应地设置应用程序或服务以支持 Windows 身份验证,例如,为了设置 Microsoft IIS 以支持 Windows 身份验证,您应该通过 dockerfile 对其进行准备:
```dockerfile
RUN Install-WindowsFeature -Name Web-Windows-Auth -IncludeAllSubFeature
RUN Import-Module WebAdministration; Set-ItemProperty 'IIS:\AppPools\SiteName' -name processModel.identityType -value 2
RUN Import-Module WebAdministration; Set-WebConfigurationProperty -Filter '/system.webServer/security/authentication/anonymousAuthentication' -Name Enabled -Value False -PSPath 'IIS:\' -Location 'SiteName'
RUN Import-Module WebAdministration; Set-WebConfigurationProperty -Filter '/system.webServer/security/authentication/windowsAuthentication' -Name Enabled -Value True -PSPath 'IIS:\' -Location 'SiteName'
```

