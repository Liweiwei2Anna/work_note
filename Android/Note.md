推送
git push origin HEAD:refs/for/master

find ./ "*.java" | xargs grep -rins "setDefaultUncaughtExceptionHandler"

Jira命令
assignee = weiwei.li AND status was in (Reopened) AND updated >= 2017-04-01 AND updated < 2017-06-30

同步且编译
repo sync -j48 && repo start master --all
repo forall -c "git reset --hard && git pull --rebase && git clean -df"

编译内核应用
source rlk_setenv.sh cx_h501_d1 && mmm frameworks/base/packages/DocumentsUI
source 之后make
source rlk_setenv.sh k8_h3722_a1 && make -j48

source tran_setenv.sh x572_h5312_a1 userdebug&& make -j16 2>&1 | tee build.log

编译powersavemanagement
source tran_setenv.sh tran_projects/x572/x572_h5312_a1
mmm packages/apps/PowersaveManagement/

K8刷机
K8
./vendor/mediatek/proprietary/scripts/sign-image/sign_image.sh
source rlk_setenv.sh k8_h3722_a1 && make -j48 2>&1 | tee build.log



断开远程ftp
net use \\192.168.10.30\Linux_user8_Samba /del /y

MTK LOG打开	*#8#9646633#*#*
adb shell am broadcast -a com.mediatek.mtklogger.ADB_CMD -e cmd_name start --ei cmd_target 23


学习
https://www.kancloud.cn/manual/thinkphp5/118003


刷IMEI
\\192.168.10.30\Linux_user6_Samba\share\AndroidO\out\target\product\k37mv1_64\vendor\etc\mddb\BPLGUInfoCustomAppSrcP_MT6735_S00_LR9_W1444_MD_LWTG_MP_W17_28_LTE_P6_1_lwg_n
\\192.168.10.30\Linux_user6_Samba\share\AndroidO\out\target\product\k37mv1_64\obj\CGEN\APDB_MT6735_S01_alps-trunk-o0.tk_W17.29


lsof -p PID 文件句柄


Google源码
http://androidxref.com/

Google 代理
10.16.13.18
8080

:set nu显示行号
：行数

/查找内容  n下一个  N上一个



MVVM    https://github.com/ittianyu/MVVM


http://192.168.10.10/#/c/176510/2


https://juejin.im/post/59ffc0c651882512a860b1b4


系统签名

1、编译android源码。
2、cd build/target/product/security/ 
3、执行 openssl pkcs8 -inform DER -nocrypt -in platform.pk8 -out platform.pem
生成platform.pem文件
4、执行 openssl pkcs12 -export -in platform.x509.pem -out platform.p12 -inkey platform.pem -password pass:huld123 -name huld
生成platform.p12文件，其中huld 为alias名（app添加签名要用到），huld123 为密码。
5、执行 keytool -importkeystore -deststorepass huld123 -destkeystore platform.jks -srckeystore platform.p12 -srcstoretype PKCS12 -srcstorepass huld123
生成platform.jks （app打签名最终用到的文件），其中-deststorepass huld123设置的是这个签名的密码，上面指令中的-src*的其他参数都是从前面两个指令中生成的。

6、将生成的platform.jks 拷贝到app工程目录下。
7、在对应需要签名的module的build.gradle中添加如下代码：
//证书信息在这里配置
signingConfigs {
    main {
        storeFile file("./key/platform.jks") //签名文件路径
        storePassword "huld123"
        keyAlias "huld"
        keyPassword "huld123"
    }
}
