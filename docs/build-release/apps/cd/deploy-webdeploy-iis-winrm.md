---
ms.assetid: 0D65C5BE-DF92-42F6-B6A4-217F0509D425
title: Deploy your Web Deploy package to IIS servers using WinRM
description: Deploy a ASP.NET or Node Web Deploy package to IIS servers from VSTS or TFS using WinRM
ms.prod: vs-devops-alm
ms.technology: vs-devops-build
ms.manager: douge
ms.author: ahomer
ms.date: 09/26/2017
---

# Deploy your Web Deploy package to IIS servers using WinRM

[!INCLUDE [temp](../../_shared/version-rm-dev14.md)]

> A simpler way to deploy web applications to IIS servers is by using [deployment groups](deploy-webdeploy-iis-deploygroups.md)
instead of WinRM. Deployment groups are currently in preview for some accounts in VSTS. They are not yet available in TFS.

Continuous deployment means starting an automated deployment process whenever a new successful build is available.
Here we'll show you how to set up continuous deployment of your ASP.NET or Node app to one or more IIS servers using Release Management.
A task running on the [Build and Release agent](../../concepts/agents/agents.md) opens a WinRM connection to each IIS server to run Powershell scripts remotely in order to deploy the Web Deploy package.

## Get set up

### Begin with a CI build

Before you begin, you'll need a CI build that publishes your Web Deploy package. To set up CI for your specific type of app, see:

* [Build your ASP.NET 4 app](../aspnet/build-aspnet-4.md)

* [Build your ASP.NET Core app](../aspnet/build-aspnet-core.md)

* [Build your Node app with Gulp](../nodejs/build-gulp.md)

### WinRM configuration

Windows Remote Management (WinRM) requires target servers to be:

* Domain-joined or workgroup-joined
* Able to communicate using the HTTP or HTTPS protocol
* Addressed by using a fully-qualified domain name (FQDN) or an IP address

This table shows the supported scenarios for WinRM.

| Joined to a | Protocol | Addressing mode |
| --------- | -------- | --------------- |
| Workgroup | HTTPS | FQDN |
| Workgroup | HTTPS | IP address |
| Domain | HTTPS | IP address |
| Domain | HTTPS | FQDN |
| Domain | HTTP | FQDN |

Ensure that your IIS servers are set up in one of these configurations.
For example, do not use WinRM over HTTP to communicate with a Workgroup machine.
Similarly, do not use an IP address to access the target server(s) when you use HTTP.
Instead, in both scenarios, use HTTPS.

Follow these steps to configure each target server.

1. Enable File and Printer Sharing. You can do this by executing the
   following command in a Command window with Administrative permissions:

   `netsh advfirewall firewall set rule group="File and Printer Sharing" new enable=yes`

1. Check your PowerShell version. You need PowerShell version
   4.0 or above installed on every target machine. To display the current
   PowerShell version, execute the following command in the PowerShell console:

   `$PSVersionTable.PSVersion`

1. Check your .NET Framework version. You need version 4.5
   or higher installed on every target machine. See
   [How to: Determine Which .NET Framework Versions Are Installed](https://msdn.microsoft.com/en-in/library/hh925568(v=vs.110).aspx).

1. Download the two scripts from [this GitHub repository](https://github.com/Microsoft/vsts-rm-documentation/tree/master/WinRM/WinRM-Http-Https-Without-Makecert)
   and copy them to every target machine. You will use them to configure WinRM in the following steps.

1. Decide if you want to use HTTP or HTTPS to communicate
   with the target machine(s).

   - If you choose HTTP, execute the following in a Command
     window with Administrative permissions:

     `ConfigureWinRM.ps1 {FQDN} http`

   >This command creates an HTTP WinRM listener and
   opens port 5985 inbound for WinRM over HTTP.

   - If you choose HTTPS, you can use either a FQDN or an IP
     address to access the target machine(s). To use a FQDN to access the target machine(s),
     execute the following in the PowerShell console with Administrative permissions:  

     `ConfigureWinRM.ps1 {FQDN} https`

     To use an IP address to access the target machine(s),
     execute the following in the PowerShell console with Administrative permissions:  

     `ConfigureWinRM.ps1 {ipaddress} https`

     >These commands create a test certificate by using
     **MakeCert.exe**, use the certificate to create
     an HTTPS WinRM listener, and open port
     5986 inbound for WinRM over HTTPS. The script also
     increases the WinRM **MaxEnvelopeSizekb** setting.
     By default on Windows Server this is 500 KB,
     which can result in a "Request size exceeded the
     configured MaxEnvelopeSize quota" error.

### IIS configuration

If you are deploying an ASP.NET app, make sure that you have ASP.NET 4.5 or ASP.NET 4.6 installed on each of your IIS target servers. For more information, see [this topic](https://www.asp.net/web-forms/overview/deployment/visual-studio-web-deployment/deploying-to-iis).

If you are deploying an ASP.NET Core application to IIS target servers, follow the additional instructions in [this topic](https://docs.microsoft.com/en-us/aspnet/core/publishing/iis) to install .NET Core Windows Server Hosting Bundle.

If you are deploying a Node application to IIS target servers, follow the instructions in [this topic](https://github.com/tjanczuk/iisnode) to install and configure IISnode on IIS servers.

In this example, we will deploy to the Default Web Site on each of the servers. If you need to deploy to another website, make sure you configure this as well.

### IIS WinRM extension

Install the [IIS Web App Deployment Using WinRM](https://marketplace.visualstudio.com/items?itemName=ms-vscs-rm.iiswebapp)
extension from Visual Studio Marketplace to your VSTS account or TFS.

## Define and test your CD release process

Continuous deployment (CD) means starting an automated release process whenever a new successful build is available. Your CD release process picks up the artifacts published by your CI build and then deploys them to your IIS servers.

1. Do one of the following:

   * If you've just completed a CI build (see above) then, in the build's
     **Summary** tab under **Deployments**, choose **Create release** followed by **Yes**.
     This starts a new release definition that's automatically linked to the build definition.

   * Open the **Releases** tab of the **Build &amp; Release** hub, open the **+** drop-down
     in the list of release definitions, and choose **Create release definition**.

1. Select the **Empty** template and click **Next**.

1. In the **Artifacts** section, make sure your CI build definition that publishes the Web Deploy package is selected as the artifact source.

1. Select the **Continuous deployment** check box, and then click **Create**.

1. On the **Variables** tab of the environment in release definition, configure a variable named **WebServers** with the list of IIS servers as its value; for example `machine1,machine2,machine3`.

1. Configure the following tasks in the environment:
  
   ![Windows Machine File Copy](../../tasks/deploy/_img/windows-machine-file-copy-icon.png) [Deploy: Windows Machine File Copy](../../tasks/deploy/windows-machine-file-copy.md) - Copy the Web Deploy package to the IIS servers.
   
   - **Source**: Select the Web deploy package (zip file) from the artifact source.
   
   - **Machines**: `$(WebServers)`
   
   - **Admin Login**: Enter the administrator credentials for the target servers. For workgroup-joined computers, use the format `.\username`. For domain-joined computers, use the format `domain\username`.
   
   - **Password**: Enter the administrator password for the target servers.
   
   - **Destination Folder**: Specify a folder on the target server where the files should be copied to.<p />
   
   ![WinRM - IIS Web App Deployment](../../tasks/deploy/_img/iis-web-application-deployment-icon.png) [Deploy: WinRM - IIS Web App Deployment](https://github.com/Microsoft/vsts-rm-extensions/blob/master/Extensions/IISWebAppDeploy/Src/Tasks/IISWebAppDeploy/README_IISAppDeploy.md) - Deploy the package.
   
   - **Machines**: `$(WebServers)`
   
   - **Admin Login**: Enter the administrator credentials for target servers. For workgroup-joined computers, use the format `.\username`. For domain-joined computers, use the format `domain\username`.
   
   - **Password**: Enter the administrator password for target servers.
   
   - **Protocol**: Select `HTTP` or `HTTPS` (depending on how you configured the target machine earlier). Note that if the target machine is workgroup-joined, you must choose `HTTPS`. You can use HTTP only if the target machine is domain-joined and configured to use a FDQN.
   
   - **Web Deploy Package**: Fully qualified path of the zip file you copied to the target server in the previous task step.
   
   - **Website Name**: `Default Web Site` (or the name of the website if you configured a different one earlier).<p />

1. Edit the name of the release definition, click **Save**, and click **OK**. Note that the default environment is named Environment1, which you can edit by clicking directly on the name.

You're now ready to create a release, which means to start the process of running the release definition with the artifacts produced by a specific build. This will result in deploying the build to IIS servers:

1. Choose **+ Release** and select **Create Release**.

1. Select the build you just completed in the highlighted drop-down list and choose **Create**.

1. Choose the release link in the popup message. For example: "Release **Release-1** has been created".

1. Open the **Logs** tab to watch the release console output.

1. After the release is complete, navigate to your site running on the IIS servers, and verify its contents.

## Q&A

<!-- BEGINSECTION class="md-qanda" -->

[!INCLUDE [temp](../../_shared/qa-versions.md)]

<!-- ENDSECTION -->

[!INCLUDE [rm-help-support-shared](../../_shared/rm-help-support-shared.md)]
