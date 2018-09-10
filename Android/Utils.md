1.输出LOG的时候，往往需要输入TAG，其实是可以省去的，通过
StackTraceElement ele = new Throwable().fillInStackTrace().getStackTrace()[index];
String.format("[%s,%d,%s] %s", ele.getFileName(), ele.getLineNumber(), ele.getMethodName(), msg);
其中index通过方法栈的深度来确定值，来输出LOG 


def SDK_VERSION = '1.2'

task makeXXXJar(dependsOn: ['compileReleaseJavaWithJavac'], type: Jar) {
    manifest {
        attributes 'Manifest-Version': SDK_VERSION,
                'Bundle-Description': 'Taichi SDK'
    }
    archiveName = "xxx-${SDK_VERSION}.jar"
    def srcClassDir = [project.buildDir.absolutePath + "/intermediates/classes/release"]
    from srcClassDir
    from(project.zipTree("libs/mmmmmm.jar"))
    exclude '**/BuildConfig.class'
    exclude '**/BuildConfig\$*.class'
    exclude "**/R.class"
    exclude "**/R\$*.class"
}

task proguardJar(dependsOn: ['makeXXXJar'], type: ProGuardTask) {
    configuration android.getDefaultProguardFile('proguard-android.txt')
    configuration project.buildDir.absolutePath + "/intermediates/proguard-rules/release/aapt_rules.txt"
    configuration 'proguard-rules.pro'
    configuration 'proguard-android.txt'
    printmapping "${project.projectDir.path}/proguard/mapping.txt"
    printseeds "${project.projectDir.path}/proguard/seeds.txt"
    printusage "${project.projectDir.path}/proguard/usage.txt"

    injars makeXXXJar.archivePath.getAbsolutePath()
    outjars makeXXXJar.archiveName

    Plugin plugin = getPlugins().hasPlugin(AppPlugin) ?
            getPlugins().findPlugin(AppPlugin) :
            getPlugins().findPlugin(LibraryPlugin)

    if (plugin != null) {
        List<String> runtimeJarList
        if (plugin.getMetaClass().getMetaMethod("getRuntimeJarList")) {
            runtimeJarList = plugin.getRuntimeJarList()
        } else if (android.getMetaClass().getMetaMethod("getBootClasspath")) {
            runtimeJarList = android.getBootClasspath()
        } else {
            runtimeJarList = plugin.getBootClasspath()
        }
        for (String runtimeJar : runtimeJarList) {
            libraryjars(runtimeJar)
        }
    }

}
