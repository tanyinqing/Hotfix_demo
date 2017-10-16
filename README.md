
[地址](https://help.aliyun.com/knowledge_detail/51418.html?spm=5176.doc51416.6.541.Vp4Z5G)

[控制台地址](https://hotfix.console.aliyun.com/?spm=5176.doc53247.2.5.KsF0HB#/list)

* 用户指南→管理后台使用说明  详细说明了后台的使用方法
* 视频专区→视频接入的详细指导
* 阿里云→产品→移动服务 各种业务都有了

* 导入3个包
* alicloud-android-utils-1.0.3.jar
* utdid4all-1.1.5.3_proguard.jar
* hotfix_core-release-3.1.3.aar

接入流程<br>
注册阿里百川开发者<br>
创建百川应用<br>
申请产品权限<br>
集成SDK<br>
生成补丁<br>
发布补丁<br>

测试的demo Hotfix_demo

生成补丁

名称 C:\Users\Administrator\Downloads 

名称 C:\Users\Administrator\Downloads\SophixPatchTool_windows\SophixPatchTool-3.1.2

- 这个是返回码的含义
- code: 1 补丁加载成功
- code: 6 服务端没有最新可用的补丁
- code: 11 RSASECRET错误，官网中的密钥是否正确请检查
- code: 12 当前应用已经存在一个旧补丁, 应用重启尝试加载新补丁
- code: 13 补丁加载失败, 导致的原因很多种, 比如UnsatisfiedLinkError等异常, 此时应该严格检查logcat异常日志
- code: 16 APPSECRET错误，官网中的密钥是否正确请检查
- code: 18 一键清除补丁
- code: 19 连续两次queryAndLoadNewPatch()方法调用不能短于3s

```
// initialize最好放在attachBaseContext最前面，初始化直接在Application类里面，切勿封装到其他类
SophixManager.getInstance().setContext(this)
                .setAppVersion(appVersion)
                .setAesKey(null)
                .setEnableDebug(true)
                .setPatchLoadStatusStub(new PatchLoadStatusListener() {
                    @Override
                    public void onLoad(final int mode, final int code, final String info, final int handlePatchVersion) {
                        // 补丁加载回调通知
                        if (code == PatchStatus.CODE_LOAD_SUCCESS) {
                            // 表明补丁加载成功
                        } else if (code == PatchStatus.CODE_LOAD_RELAUNCH) {
                            // 表明新补丁生效需要重启. 开发者可提示用户或者强制重启;
                            // 建议: 用户可以监听进入后台事件, 然后调用killProcessSafely自杀，以此加快应用补丁，详见1.3.2.3
                        } else {
                            // 其它错误信息, 查看PatchStatus类说明
                        }
                    }
                }).initialize();
                
  //查看并下载网络补丁             
// queryAndLoadNewPatch不可放在attachBaseContext 中，否则无网络权限，建议放在后面任意时刻，如onCreate中
SophixManager.getInstance().queryAndLoadNewPatch();
```
1. 接口说明  
- queryAndLoadNewPatch方法拉取补丁。这个方法调用需要尽可能的早, 推荐在Application的onCreate方法中调用
2. SophixManager.getInstance().setContext(this) 方法说明
- setSecretMetaData(idSecret, appSecret, rsaSecret): <可选，推荐使用> 三个Secret分别对应AndroidManifest里面的三个，可以不在AndroidManifest设置而是用此函数来设置Secret。放到代码里面进行设置可以自定义混淆代码，更加安全，此函数的设置会覆盖AndroidManifest里面的设置，如果对应的值设为null，默认会在使用AndroidManifest里面的。
- setEnableDebug(isEnabled): <可选> isEnabled默认为false, 是否调试模式, 调试模式下会输出日志以及不进行补丁签名校验. 线下调试此参数可以设置为true, 查看日志过滤TAG:Sophix, 同时强制不对补丁进行签名校验, 所有就算补丁未签名或者签名失败也发现可以加载成功. 但是正式发布该参数必须为false, false会对补丁做签名校验, 否则就可能存在安全漏洞风险
- setAesKey(aesKey): <可选> 用户自定义aes秘钥, 会对补丁包采用对称加密。这个参数值必须是16位数字或字母的组合，是和补丁工具设置里面AES Key保持完全一致, 补丁才能正确被解密进而加载。此时平台无感知这个秘钥, 所以不用担心阿里云移动平台会利用你们的补丁做一些非法的事情。
- setPatchLoadStatusStub(new PatchLoadStatusListener()): <可选> 设置patch加载状态监听器, 该方法参数需要实现PatchLoadStatusListener接口, 接口说明见1.3.2.2说明

1.3.2.5 PatchLoadStatusListener接口

该接口需要自行实现并传入initialize方法中, 补丁加载状态会回调给该接口, 参数说明如下:

- mode: 补丁模式, 0:正常请求模式 1:扫码模式 2:本地补丁模式
- code: 补丁加载状态码, 详情查看PatchStatusCode类说明
- info: 补丁加载详细说明, 详情查看PatchStatusCode类说明
- handlePatchVersion: 当前处理的补丁版本号, 0:无 -1:本地补丁 其它:后台补丁

1.4 版本管理说明

说明一：patch是针对客户端具体某个版本的，patch和具体版本绑定

eg. 应用当前版本号是1.1.0, 那么只能在后台查询到1.1.0版本对应发布的补丁, 而查询不到之前1.0.0旧版本发布的补丁.
说明二：针对某个具体版本发布的新补丁, 必须包含所有的bugfix, 而不能依赖补丁递增修复的方式, 因为应用仅可能加载一个补丁

eg. 针对1.0.0版本在后台发布了一个补丁版本号为1的补丁修复了bug1, 然后发现此时针对这个版本补丁1修复的不完全, 代码还有bug2, 在后台重新发布一个补丁版本号为2的补丁, 那么此时补丁2就必须同时包含bug1和bug2的修复才行, 而不是只包含bug2的修复(bug1就没被修复了)