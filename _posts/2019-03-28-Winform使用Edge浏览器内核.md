---
layout:     post
title:      Winform使用Edge浏览器内核
subtitle:   摆脱webbroswer的禁锢
date:       2019-03-28
author:     skywolf627
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - Winform
    - C#
---

## 前言
经常写点winform小工具，但是winform的控件又不好看，所以一般都是在winform中集成一个浏览器控件用来显示HTML内容，然后通过JS和winform进行交互。这样既美观也方便

## 困境
由于winform的webbroswer控件使用的是低版本的IE内核，不支持H5，css3。这样兼容性很难保证。

## 解决办法
  ### 1、修改注册表 
> 通过修改注册表设置当前程序的浏览器模式

> 优点：简单方便，无需额外引用

> 缺点：兼容性差，如果电脑未装高版本的IE则无用
  
        /// <summary>  
        /// 修改注册表信息来兼容当前程序  
        ///   
        /// </summary>  
        static void SetWebBrowserFeatures(int ieVersion)
        {
            // don't change the registry if running in-proc inside Visual Studio  
            if (LicenseManager.UsageMode != LicenseUsageMode.Runtime)
                return;
            //获取程序及名称  
            var appName = System.IO.Path.GetFileName(System.Diagnostics.Process.GetCurrentProcess().MainModule.FileName);
            //得到浏览器的模式的值  
            UInt32 ieMode = GeoEmulationModee(ieVersion);
            var featureControlRegKey = @"HKEY_CURRENT_USER\Software\Microsoft\Internet Explorer\Main\FeatureControl\";
            //设置浏览器对应用程序（appName）以什么模式（ieMode）运行  
            Registry.SetValue(featureControlRegKey + "FEATURE_BROWSER_EMULATION",
                appName, ieMode, RegistryValueKind.DWord);
        }
        /// <summary>  
        /// 获取浏览器的版本  
        /// </summary>  
        /// <returns></returns>  
        static int GetBrowserVersion()
        {
            int browserVersion = 0;
            using (var ieKey = Registry.LocalMachine.OpenSubKey(@"SOFTWARE\Microsoft\Internet Explorer",
                RegistryKeyPermissionCheck.ReadSubTree,
                System.Security.AccessControl.RegistryRights.QueryValues))
            {
                var version = ieKey.GetValue("svcVersion");
                if (null == version)
                {
                    version = ieKey.GetValue("Version");
                    if (null == version)
                        throw new ApplicationException("Microsoft Internet Explorer is required!");
                }
                int.TryParse(version.ToString().Split('.')[0], out browserVersion);
            }
            //如果小于7  
            if (browserVersion < 7)
            {
                throw new ApplicationException("不支持的浏览器版本!");
            }
            return browserVersion;
        }
        /// <summary>  
        /// 通过版本得到浏览器模式的值  
        /// </summary>  
        /// <param name="browserVersion"></param>  
        /// <returns></returns>  
        static UInt32 GeoEmulationModee(int browserVersion)
        {
            UInt32 mode = 11000; // Internet Explorer 11. Webpages containing standards-based !DOCTYPE directives are displayed in IE11 Standards mode.   
            switch (browserVersion)
            {
                case 7:
                    mode = 7000; // Webpages containing standards-based !DOCTYPE directives are displayed in IE7 Standards mode.   
                    break;
                case 8:
                    mode = 8000; // Webpages containing standards-based !DOCTYPE directives are displayed in IE8 mode.   
                    break;
                case 9:
                    mode = 9000; // Internet Explorer 9. Webpages containing standards-based !DOCTYPE directives are displayed in IE9 mode.                      
                    break;
                case 10:
                    mode = 10000; // Internet Explorer 10.  
                    break;
                case 11:
                    mode = 11000; // Internet Explorer 11  
                    break;
            }
            return mode;
        }
  

### 2、集成Chrome内核（CefSharp或者WebKit.net）
  > 需要下载引用文件
  
  >优点：兼容性好，完美支持H5,CSS3
  
  >缺点：需要额外引用第三方DLL文件，体积很大，对于做个小工具可能1M不到，引用文件却几十M有点无法接受
      
      CefSharp需要.NET 4.5.2或以上，不支持Any CPU,所以生成的时候必须选择目标平台（x86或x64）
           
        
### 3、使用Edge内核（个人推荐）
  > 虽然微软的浏览器很多让人吐槽的地方，不过集成到winform中使用还是够的。
  
  > 优点：兼容性好，支持H5,CSS3
  
  > 缺点：必须.NET 4.6.2 或以上，VS2017, Win10 17110或更高版本
  
      下载应用包
      Nuget Microsoft.Toolkit.Forms.UI.Controls.WebView
      
      隐藏引用DLL （可以跳过，这个DLL能生成单EXE程序，不会显示一堆DLL）
      Nuget Costura.Fody   （最新版3.3.3 **下载1.6.2** ）
      
      将控件拖到winform上
      //允许JS交互
      webView1.IsScriptNotifyAllowed = true;
      //页面加载完毕事件
      webView1.NavigationCompleted += (s, e) =>
      {
          //TODO something
      };
      //JS调用WINFORM事件，统一在此事件进行处理
      webView1.ScriptNotify += (s, html) =>
      {
          //JS调用winform事件的时候 可以传入json字符串，这样便于扩展
          switch(html)
          {
              case "AAA":              
              //TODO something
              break;
              case "BBB":
              //TODO something
              break;
          }
      };
      
      //跳转页面或加载HTML
       webView1.Navigate("http://www.baidu.com");
       //调用JS函数   多个参数使用  new {"参数1","参数2","参数3"}
       webView1.InvokeScript("JS函数名", "JS参数");         
       //利用JS获取网页源码
       webView1.InvokeScript("eval", "window.external.notify(document.body.innerHTML);");
       
       //JS调用winform
       window.external.notify("参数");
       
## 结尾
这三个方式我都使用过，我比较喜欢第三种方式，因为一般写的程序都是自用，所以安装.net 和win10版本都不是问题，果断使用第三种方式。
