# jni层验证签名和包名  
```c++
#include <jni.h>
#include <string>
#include <string.h>
#include <android/log.h>
#include <syscall.h>

#define   LOG_TAG    "native_log"
#define   LOGD(...)  __android_log_print(ANDROID_LOG_DEBUG,LOG_TAG,__VA_ARGS__)
#define   LOGI(...)  __android_log_print(ANDROID_LOG_INFO,LOG_TAG,__VA_ARGS__)
#define   LOGE(...)  __android_log_print(ANDROID_LOG_ERROR,LOG_TAG,__VA_ARGS__)

//合法的APP包名
const char *app_packageName = "com.test";
//合法的hashcode
const int app_signature_hash_code = 12345642;

extern "C" JNIEXPORT jstring


JNICALL
Java_im_huochat_com_myapplication_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}



/**
* 校验APP 包名和签名是否合法 返回值为1 表示合法
*/
jint checkSignature(JNIEnv *env) {

    //获取Activity Thread的实例对象
    jclass activityThread = env->FindClass("android/app/ActivityThread");
    jmethodID currentActivityThread = env->GetStaticMethodID(activityThread, "currentActivityThread", "()Landroid/app/ActivityThread;");
    jobject at = env->CallStaticObjectMethod(activityThread, currentActivityThread);
    //获取Application，也就是全局的Context
    jmethodID getApplication = env->GetMethodID(activityThread, "getApplication", "()Landroid/app/Application;");
    jobject context = env->CallObjectMethod(at, getApplication);

    //Context的类
    jclass context_clazz = env->GetObjectClass(context);
    // 得到 getPackageManager 方法的 ID
    jmethodID methodID_getPackageManager = env->GetMethodID(context_clazz, "getPackageManager", "()Landroid/content/pm/PackageManager;");

    // 获得PackageManager对象
    jobject packageManager = env->CallObjectMethod(context, methodID_getPackageManager);
//	// 获得 PackageManager 类
    jclass pm_clazz = env->GetObjectClass(packageManager);
    // 得到 getPackageInfo 方法的 ID
    jmethodID methodID_pm = env->GetMethodID(pm_clazz, "getPackageInfo", "(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;");
//
//	// 得到 getPackageName 方法的 ID
    jmethodID methodID_pack = env->GetMethodID(context_clazz, "getPackageName", "()Ljava/lang/String;");

    // 获得当前应用的包名
    jobject application_package = env->CallObjectMethod(context, methodID_pack);

    // 获得PackageInfo
    jobject packageInfo = env->CallObjectMethod(packageManager, methodID_pm, application_package, 64);

    jclass packageinfo_clazz = env->GetObjectClass(packageInfo);

    jfieldID fieldID_signatures = env->GetFieldID(packageinfo_clazz, "signatures", "[Landroid/content/pm/Signature;");
    jobjectArray signature_arr = (jobjectArray)env->GetObjectField(packageInfo, fieldID_signatures);

    //Signature数组中取出第一个元素
    jobject signature = env->GetObjectArrayElement(signature_arr, 0);
    //读signature的hashcode
    jclass signature_clazz = env->GetObjectClass(signature);
    jmethodID methodID_hashcode = env->GetMethodID(signature_clazz, "hashCode", "()I");
    jint hashCode = env->CallIntMethod(signature, methodID_hashcode);
    LOGE("hashcode: %d\n", hashCode);

    const char *p = env->GetStringUTFChars((jstring)application_package, false);

    if (strcmp(p, app_packageName) != 0) {
        return -1;
    }
    if (hashCode != app_signature_hash_code) {
        return -2;
    }
    return 1;
}


// JNI_OnLoad函数实现
jint JNI_OnLoad(JavaVM* vm, void* reserved){
    JNIEnv* env = NULL;
    jint result = -1;

    if (vm->GetEnv((void**) &env, JNI_VERSION_1_6) != JNI_OK) {
        return -1;
    }

    if(checkSignature(env) != 1){
        throw 'signature wrong';
    }

    //一定要返回版本号，否则会出错。
    result = JNI_VERSION_1_6;
    return result;
}

```