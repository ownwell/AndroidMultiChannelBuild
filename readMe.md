# 使用原理
这个脚本是参考美团技术博客中提到的在apk文件添加一个渠道文件名的方法，具体文章见本人的博客：[Android的快速多渠道打包](http://ownwell.github.io/2015/09/28/mutichannel4Android/)。

# 使用方法
## 打包
   
   一个已经打包的母包，通过Python脚本和我们的渠道信息，会生成各个渠道。100个渠道，这个操作耗时不到20s，除了生成母包耗时外，生成渠道包，是瞬间的！！！！！！
    
    母包：没有渠道信息的包
    子包:通过母包打出渠道信息的包，其实就是再母包的apk的`MATE—INF下`新建一个空文件，文件名称有渠道信息。
   
   
   
将一个已经签名过的母包（现在也支持任意的渠道包），放到项目的根目录下。
文件结构


|---**.apk    
|---MultiChannelBuildTool.py    
|---info    
|------channel.txt   

其中channel.txt放的就是我们自己的渠道，每个渠道记得换行，增删都是添加或删除一行。

直接运行这个python脚本（python MultiChannelBuildTool.py），这个目录下会生成包含了各个渠道包的文件夹。

对于第一次使用这个方式的人，一定要记得去检查是否生成了各个渠道，需要的话可以先自己用apktool自己反编译看看是不是再Meta-INF文件下有`channel_渠道名称`这个文件。若有就OK，说明Everything in control，若没有记得查看原因。

    改进的python脚本中，可以使用任何渠道包，在生成子包过程中，发现母包带有渠道信息，就会将母包的渠道文件给删除，再重新生成一个干净的、没有渠道信息母包，然后按照以前的方式生成各个渠道包。
    


## 读取

每次我们统计渠道都要自己从apk文件中读取，然后再UMeng统计中手动设置渠道名称（一般的第三方统计都支持这个的）。

从apk读取渠道名称，思路是第一次将渠道名称放到本地SP中，下次读取就直接读取本地的SP渠道信息，但是升级时，可能需要更换渠道信息（我1.0从豌豆荚下的，1.1可能从Google市场下的哦）。

```Java

import android.app.Activity;
import android.content.Context;
import android.content.SharedPreferences;
import android.content.pm.ApplicationInfo;


import java.io.IOException;
import java.util.Enumeration;
import java.util.zip.ZipEntry;
import java.util.zip.ZipFile;

/**
 * @author Cyning
 * @since 2015.09.17
 * Time   下午9:07
 * Desc   <p>多渠道打包，获得包的渠道，为了能够能提高效率，在使用过程中，
 * 只有新版本才进行从从Zip文件中获得安装包的渠道名称，否则都是读取本地缓存的渠道名称</p>
 * <p>
 * 已经总结为博客，博客链接：http://ownwell.github.io/2015/09/28/mutichannel4Android/
 *
 * </p>
 */

public class MutiChannelConfig {

  public static final String Version = "version";

  public static final String Channel = "channel";

  public static final String DEFAULT_CHANNEL = "";//默认渠道自己的官方渠道

  public static final String Channel_File = "channel";

  /**
   * 从Zip文件中获得安装包的渠道名称
   */
  public static String getChannelFromMeta(Context context) {
    ApplicationInfo appinfo = context.getApplicationInfo();
    String sourceDir = appinfo.sourceDir;
    String ret = "";
    ZipFile zipfile = null;
    try {
      zipfile = new ZipFile(sourceDir);
      Enumeration<?> entries = zipfile.entries();
      while (entries.hasMoreElements()) {
        ZipEntry entry = ((ZipEntry) entries.nextElement());
        String entryName = entry.getName();
        if (entryName.startsWith("META-INF") && entryName.contains("channel_")) {
          ret = entryName;
          break;
        }
      }
    } catch (IOException e) {
      e.printStackTrace();
    } finally {
      if (zipfile != null) {
        try {
          zipfile.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
    String[] split = ret.split("_");
    if (split != null && split.length >= 2) {
      return ret.substring(split[0].length() + 1);
    } else {
      return DEFAULT_CHANNEL;
    }
  }

  /**
   * 得到渠道名
   * <p>当时新版本时，就从zip文件中获取，同时保存到本地</p>
   * <p>当不是新版本时，就直接读取本地缓存的文件</p>
   */
  public static String getChannel(Context mContext) {
    String channel = DEFAULT_CHANNEL;
    if (isNewVersion(mContext)) {//是新版本
      channel = getChannelFromMeta(mContext);
      LayzLog.d("new Version  %s", channel);
      saveChannel(mContext, channel);//保存当前版本
    } else {
      channel = getCachChannel(mContext);
    }
    return channel;
  }

  /**
   * 保存当前的版本号和渠道名
   */
  public static void saveChannel(Context mContext, String channel) {
    SharedPreferences mSettinsSP =
        mContext.getSharedPreferences(Channel_File, Activity.MODE_PRIVATE);
    SharedPreferences.Editor mSettinsEd = mSettinsSP.edit();
    mSettinsEd.putString(Version, APPUtils.getAppVersion(mContext));
    mSettinsEd.putString(Channel, channel);
    //提交保存
    mSettinsEd.commit();
  }

  /**
   * 判断一下本地缓存的版本号和现在运行的版本号是否一致
   */
  private static boolean isNewVersion(Context mContext) {
    SharedPreferences mSettinsSP =
        mContext.getSharedPreferences(Channel_File, Activity.MODE_PRIVATE);
    String version = APPUtils.getAppVersion(mContext);// 得到apk的版本信息
    LayzLog.d("version%s", version);
    return !mSettinsSP.getString(Version, "").equals(version);
  }

  /**
   * 读取本地缓存的版本号
   */
  private static String getCachChannel(Context mContext) {
    SharedPreferences mSettinsSP =
        mContext.getSharedPreferences(Channel_File, Activity.MODE_PRIVATE);
    return mSettinsSP.getString(Channel, DEFAULT_CHANNEL);
  }
}
```

每次都通过`getChannel`获取渠道，再调用第三方统计设置渠道的方法，就Ok。打包节省了您的一大笔时间，你就偷着了吧。




