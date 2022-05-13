1. 将 pagefile.sys 从C盘移动到D盘。配置位置：电脑 - 属性 - 高级系统设置 - 高级 - 性能设置 - 高级 - 更改：取消勾选，C盘改为无分页文件，D盘改为系统管理的大小，重启电脑即可。
   
   参考：[win10的pagefile.sys是什么文件？pagefile.sys文件太大如何移动到D盘中？](https://blog.csdn.net/xrinosvip/article/details/81352823)

2. 移动 nuget 缓存到D盘。nuget 缓存在以下位置。将 %UserProfile%.nuget\packages 移动到D盘自定义目录。
   
   - %LocalAppData%\NuGet\Cache
   - %UserProfile%\.nuget\packages
   
   然后打开 %AppData%\NuGet\NuGet.Config ，在这个文件夹添加下面代码
   
   ```xml
   <configuration>
     <config>
        <add key="globalPackagesFolder" value="D:\lindexi\packages" />
     </config>
   </configuration>
   ```
   
   参考：[(59条消息) 如何移动 nuget 缓存文件夹_weixin_34409741的博客-CSDN博客](https://blog.csdn.net/weixin_34409741/article/details/89770528)
