����
git push origin HEAD:refs/for/master

find ./ "*.java" | xargs grep -rins "setDefaultUncaughtExceptionHandler"

Jira����
assignee = weiwei.li AND status was in (Reopened) AND updated >= 2017-04-01 AND updated < 2017-06-30

ͬ���ұ���
repo sync -j48 && repo start master --all
repo forall -c "git reset --hard && git pull --rebase && git clean -df"

�����ں�Ӧ��
source rlk_setenv.sh cx_h501_d1 && mmm frameworks/base/packages/DocumentsUI
source ֮��make
source rlk_setenv.sh k8_h3722_a1 && make -j48

source tran_setenv.sh x572_h5312_a1 userdebug&& make -j16 2>&1 | tee build.log

����powersavemanagement
source tran_setenv.sh tran_projects/x572/x572_h5312_a1
mmm packages/apps/PowersaveManagement/

K8ˢ��
K8
./vendor/mediatek/proprietary/scripts/sign-image/sign_image.sh
source rlk_setenv.sh k8_h3722_a1 && make -j48 2>&1 | tee build.log



�Ͽ�Զ��ftp
net use \\192.168.10.30\Linux_user8_Samba /del /y

MTK LOG��	*#8#9646633#*#*
adb shell am broadcast -a com.mediatek.mtklogger.ADB_CMD -e cmd_name start --ei cmd_target 23


ѧϰ
https://www.kancloud.cn/manual/thinkphp5/118003


ˢIMEI
\\192.168.10.30\Linux_user6_Samba\share\AndroidO\out\target\product\k37mv1_64\vendor\etc\mddb\BPLGUInfoCustomAppSrcP_MT6735_S00_LR9_W1444_MD_LWTG_MP_W17_28_LTE_P6_1_lwg_n
\\192.168.10.30\Linux_user6_Samba\share\AndroidO\out\target\product\k37mv1_64\obj\CGEN\APDB_MT6735_S01_alps-trunk-o0.tk_W17.29


lsof -p PID �ļ����


GoogleԴ��
http://androidxref.com/

Google ����
10.16.13.18
8080

:set nu��ʾ�к�
������

/��������  n��һ��  N��һ��



MVVM    https://github.com/ittianyu/MVVM


http://192.168.10.10/#/c/176510/2


https://juejin.im/post/59ffc0c651882512a860b1b4


status:merged AND owner:"Ji Tingting(��͢͢) <tingting.ji@reallytek.com>"  OR owner:"Xu Xiaodong(������) <xiaodong.xu@reallytek.com>"  OR owner:"Yang Bincai�����ţ� <bincai.yang@reallytek.com>" OR owner:"Li Jun(���) <jun.li@reallytek.com>"   OR owner:"Zhang Guoliang(�Ź���) <guoliang.zhang@reallytek.com>" OR owner:"Song Zeyou(������) <zeyou.song@reallytek.com>" OR owner:"Yu Qiwen(������) <qiwen.yu@reallytek.com>"  OR  owner:"Xie Zheng(л��) <zheng.xie@reallytek.com>"  OR owner:"Shi Xinxin(ʷ����) <xinxin.shi@reallytek.com>"


ϵͳǩ��


1������androidԴ�롣
2��cd build/target/product/security/ 
3��ִ�� openssl pkcs8 -inform DER -nocrypt -in platform.pk8 -out platform.pem
����platform.pem�ļ�
4��ִ�� openssl pkcs12 -export -in platform.x509.pem -out platform.p12 -inkey platform.pem -password pass:huld123 -name huld
����platform.p12�ļ�������huld Ϊalias����app���ǩ��Ҫ�õ�����huld123 Ϊ���롣
5��ִ�� keytool -importkeystore -deststorepass huld123 -destkeystore platform.jks -srckeystore platform.p12 -srcstoretype PKCS12 -srcstorepass huld123
����platform.jks ��app��ǩ�������õ����ļ���������-deststorepass huld123���õ������ǩ�������룬����ָ���е�-src*�������������Ǵ�ǰ������ָ�������ɵġ�

6�������ɵ�platform.jks ������app����Ŀ¼�¡�
7���ڶ�Ӧ��Ҫǩ����module��build.gradle��������´��룺
//֤����Ϣ����������
signingConfigs {
    main {
        storeFile file("./key/platform.jks") //ǩ���ļ�·��
        storePassword "huld123"
        keyAlias "huld"
        keyPassword "huld123"
    }
}