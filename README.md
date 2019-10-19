## **深入探索Android热修复技术原理**

去年一整年android社区中刮过了一阵热修复的风，各大厂商，逼格大牛纷纷开源了热修复框架，产品过程中怎么可能没有bug呢？重新打包上线？成本太高用户体验也不好，咋办？上热修复呗。

好吧，既然要开始上热修复的功能，那么就得调研一下热修复的原理。下面我将分别讲述一下热修复的原理，各大热修复框架的比较，以及自身产品中热修复功能的实践。

一、什么是热修复？

## 正常开发流程

![823551-20180311132422132-630335315](D:\360MoveData\Users\Administrator\Desktop\823551-20180311132422132-630335315.png)

## 热修复开发流程

![823551-20180311132434276-1651642452](D:\360MoveData\Users\Administrator\Desktop\823551-20180311132434276-1651642452.png)

## 热修复优势

![823551-20180311132444619-1108391379](D:\360MoveData\Users\Administrator\Desktop\823551-20180311132444619-1108391379.png)

## 修复什么？

![823551-20180311132457465-786355225](D:\360MoveData\Users\Administrator\Desktop\823551-20180311132457465-786355225.png)热修复的原理

1. 热修复原理
2. 通过更改dex加载顺序实现热修复 最新github上开源了很多热补丁动态修复框架，大致有：[HotFix](https://github.com/dodola/HotFix)      [Nuwa](https://github.com/jasonross/Nuwa)      [DroidFix](https://github.com/bunnyblue/DroidFix) 上述三个框架呢，根据其描述，原理都来自： [安卓App热补丁动态修复技术介绍](https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a&scene=1&srcid=1106Imu9ZgwybID13e7y2nEi#wechat_redirect)，以及[Android dex分包方案](http://my.oschina.net/853294317/blog/308583)，其核心原理就是通过更改含有bug的dex文件的加载顺序。在dex的加载中，若以找到方法则不会继续查找，所以如果能让修复之后的方法在含有bug的方法之前加载就能达到修复bug的目的，而这个框架都是基于这个思路实现的，当然了具体实现过程中有很多坑，大家可以参考一下上面提到的两篇文章。
3. 通过Native替换方法指针的方式实现热修复 这里主要是阿里开源的两个热修复框架：[Dexpost](https://github.com/alibaba/dexposed)        [AndFix](https://github.com/alibaba/AndFix)都是通过Native层使用指针替换的方法替换bug，达到修复bug的目的的。

2、**热修复框架的对比**

热修复框架的种类繁多，按照公司团队划分主要有以下几种：

![1571488748901](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1571488748901.png)

虽然热修复框架很多，但热修复框架的核心技术主要有三类，分别是代码修复、资源修复和动态链接库修复，其中每个核心技术又有很多不同的技术方案，每个技术方案又有不同的实现，另外这些热修复框架仍在不断的更新迭代中，可见热修复框架的技术实现是繁多可变的。作为开发需需要了解这些技术方案的基本原理，这样就可以以不变应万变。

部分热修复框架的对比如下表所示
![1571488793012](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1571488793012.png)



3、热修复实践

最终我们App的热修复方案选择的是AndFix，原因有三： （1）AndFix支持android2.3-6.0，所以在机型上的是适配上是没问题的； （2）AndFix是由阿里开源的，并且持续维护中，目前不少公司已经使用其作为自身App的热修复方案； （3）通过修改Dex加载顺序的方式实现热修复需要重新启动App，并且相应的开源框架多多少少存在着问题，没有持续的维护；

因此我们最终选择了AndFix作为我们的开源方案。具体的AndFix集成方式可参考[github中AndFix的介绍](https://github.com/alibaba/AndFix)

这里简单介绍一下具体的继承流程 （1）在App的Application的onCreate方法中执行AndFix的初始化操作； （2）判断服务器端是否有可更新的热修复差异包 （3）若无则直接退出，若有则下载并执行修复动作 （4）修复完成之后删除下载的补丁差异包 （5）在判断服务器端是否有可更新的补丁包的时候可添加灰度，如版本，渠道，用户等，实现对补丁包定制化的修复

另外需要说明的是：若一个版本中存在着多个bug，则一般的都是让后一个补丁包覆盖前一个补丁包，并删除前一个补丁包，简单来说就是对于每一个版本至多有一个补丁包。

最后贴上App端AndFix的实现源码：

```
/**
 * Created by aaron on 2016/3/7.
 * 主要用于实现热修复逻辑
 *   采用阿里巴巴开源框架-andfix
 *
 */
public class AndfixManager {
    public static final String TAG = AndfixManager.class.getSimpleName();

    // AndfixManager单例对象
    private static AndfixManager instance = null;
    // 补丁文件名称
    public static final String PATCH_FILENAME = "/patchname.apatch";

    public static PatchManager patchManager = null;
    private AndfixManager() {}

    /**
     * 线程安全之懒汉模式实现单例模型
     * @return
     */
    public static synchronized AndfixManager getInstance() {
        return instance == null ? new AndfixManager() : instance;
    }

    /**
     * 执行andfix初始化操作
     */
    public static void init(Context mContext) {
        if (mContext == null) {
            L.i("初始化热修复框架，参数错误！！！");
            return;
        }
        patchManager = new PatchManager(mContext);
        // 初始化patch版本，这里初始化的是当前的App版本；
        patchManager.init(VersionUtils.getVersionName(mContext));
        // 加载已经添加到PatchManager中的patch
        patchManager.loadPatch();


        downLoadAndAndPath(mContext);

    }

    /**
     * 请求服务器获取补丁文件并加载
     */
    public static void downLoadAndAndPath(final Context mContext) {
        // 请求服务器获取差异包
        ExtInterface.GetShContent.Request.Builder request = ExtInterface.GetShContent.Request.newBuilder();

        // 获取本地保存的补丁包版本号
        final String patchVersion = AndfixSp.getPatchVersion(mContext);
        L.i(TAG, "patchVersion:" + patchVersion);
        if (!TextUtils.isEmpty(patchVersion)) {
            request.setShVersion(patchVersion);
        } else {
            request.setShVersion("0");
        }
        NetworkTask task = new NetworkTask(Cmd.CmdCode.GetShContent_SSL_VALUE);
        task.setBusiData(request.build().toByteArray());
        NetworkUtils.executeNetwork(task, new HttpResponse.NetWorkResponse<UUResponseData>() {
            @Override
            public void onSuccessResponse(UUResponseData responseData) {
                if (responseData.getRet() == 0) {
                    try {
                        ExtInterface.GetShContent.Response response = ExtInterface.GetShContent.Response.parseFrom(responseData.getBusiData());
                        // 若返回成功，则更新脚本下载补丁包
                        if (response.getRet() == 0) {
                            ByteString zipDatas = response.getContent();
                            // 数据解压缩
                            byte[] oriDatas = GZipUtils.decompress(zipDatas.toByteArray());
                            String patchFileName = mContext.getCacheDir() + PATCH_FILENAME;
                            L.i(TAG, "patchFileName:" + response.getShVersion());
                            // 将byte数组数据写入文件
                            boolean boolResult = getFileFromBytes(patchFileName, oriDatas);
                            // 写入文件成功则加载
                            if (boolResult) {
                                patchManager.removeAllPatch();
                                patchManager.addPatch(patchFileName);

                                // 保存补丁版本号
                                AndfixSp.putPatchVersion(mContext, response.getShVersion());
                                // 删除补丁文件
                                File files = new File(patchFileName);
                                if (files.exists()) {
                                    files.delete();
                                }
                            }

                        } else {
                            // -1 请求失败
                            // 1 请求成功，但是没有更新版本的脚本
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }

            @Override
            public void onError(VolleyError errorResponse) {
            }

            @Override
            public void networkFinish() {
            }
        });

    }


    /**
     * 根据数组获取文件
     * @param path
     * @param oriDatas
     */
    public static boolean getFileFromBytes(String path, byte[] oriDatas) {
        boolean result = false;
        if (TextUtils.isEmpty(path)) {
            return result;
        }
        if (oriDatas == null || oriDatas.length == 0) {
            return result;
        }

        try {
            FileOutputStream fos = new FileOutputStream(path);
            fos.write(oriDatas);
            fos.close();
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }

    }
}
```

**总结：** android的热修复原理大体上分为两种，其一是通过dex的执行顺序实现Apk热修复的功能，但是其需要将App重启才能生效，其二是通过Native修改函数指针的方式实现热修复，有兴趣的同学可以深入研究。

详情可点击热修复视频教程https://www.bilibili.com/video/av71274292
