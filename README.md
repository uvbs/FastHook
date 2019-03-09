原创]FastHook——一种高效稳定、简洁易用的Android Hook框架
5小时前  190
图灵技师  
 举报
一、前言
GitHub：FastHook

在使用YAHFA框架的过程中，遇到了些问题，为了解决这些问题在YAHFA的基础上写了FastHook框架。本文分析内容基于Android 8.1。

二、YAHFA
首先我们来看看YAHFA框架基本流程，再分析其实现原理。


Target方法EntryPoint替换为HookTrampoline。当执行Target方法时，实际执行HookTrampoline，从而实现方法Hook。
HookTrampoline先将r0设置为Hook方法，再跳转到Hook方法EntryPoint执行Hook方法。

Hook方法参数与Target方法参数须一致，在Hook方法里调用Backup方法达到间接调用Target方法的目的。

Backup方法必须是static方法（如果Target方法不是static方法，Backup方法第一个参数必须为this，Hook方法不需要一定是static方法，只要保证参数一致即可）。static方法是静态分派的，这可以保证调用的是Backup方法本身。Backup方法必须要完全备份Target方法，ART需要知道native code与dex code的映射关系，例如一条native指令对应哪条dex指令，这个映射关系需要EntryPoint来计算，而为了实现Hook，我们替换了Target方法的EntryPoint。所以我们必须完全复制Backup方法，此时我们执行的还是Backup方法，只是这个Backup方法的内容跟Target完全一样，这样间接达到调用Target方法的目的。

存在的问题： 
性能损耗大。设置 kAccCompileDontBother禁止了JIT和AOT编译 。
Backup方法无法被再次解析。而8.0之后， dex_cache_resolved_methods_只有1024大小，方法索引为 dex_method_index_%1024 ， Backup方法所在dex方法数超过1024时，Backup方法有可能被覆盖，触发方法解析 。
Moving GC问题。当Target方法Class对象被移动时， 由于Backup方法是备份得到的，不会更新到新地址， 这就造成空指针异常 。
方法内联导致Hook失效。 
三、FastHook
FastHook提供了两种方案，一种类似Native Inline Hook，另一个依旧是Entrypoint替换。 
Inline模式

EntryPoint替换模式

Inline模式实现
用一段跳转代码JumpTrampoline覆盖Target方法native code头几个字节，这样当执行Target方法时会跳转到我们指定地方，到达Hook的目的。 JumpTrampoline跳转到HookTrampoline， HookTrampoline里判断当前方法是否是需要Hook的方法（相似的方法可能会复用同一native code），如果是，将r0设置为Hook方法并跳转到Hook方法EntryPoint，反之跳到下一个HookTrampoline，最后一个Trampoline会负责执行回原方法，先执行被JumpTrampoline覆盖掉的指令，再跳转到原方法下一条指令。 Backup方法不再备份Target方法，这里命名为Forward方法，这是一个静态native方法，一方面可以阻止被内联，另一方面也表明该方法仅仅起到forward作用，没有方法体，也不会被调用。Forward方法Entrypoint替换为TargetTrampoline，该Trampoline实现调用原方法逻辑。这样便可以完美解决方法解析与Moving GC问题。 
EntryPoint模式实现
Hook部分与YAHFA一致，调用原方法部分与Inline模式类似，使用替换Forward方法EntryPoint实现，不过这里替换EntryPoint为art_quick_to_interpreter_bridge，这样既能解决方法解析与Moving GC，也能规避native code与dex code映射的问题。 
方法编译
为了实现Inline Hook，需要一种方法将方法编译成本地代码。这里用libart-compiler.so提供的jit_compile_method方法进行编译，在7.0之后Android限制了dlopen等函数的调用，这里用fake_dlfcn代替。有一点需要注意，不要用当前线程编译，JIT编译时会改变Thread状态，checkpoint时有概率会crash，这里新起一个线程用来编译。 
Hook安全
下列情况下可能会改变方法EntryPoint：
文件加载
类初始化
JIT编译
JIT垃圾回收（类似Mark-Sweep，设置为QuickToInterpreterBridge）
解释执行（如果存在JIT入口则设置为JIT入口 ）
无论是Inline模式还是EntryPoint替换模式，都要避免EntryPoint被改变。因此在Hook之前就需要判断JIT状态，如果当前方法正在JIT编译或者等待JIT编译，就需要等待JIT编译完成，如果是未编译或已编译完成，可直接Hook。另外，Inline模式需要覆盖原方法指令，因此就需要保证在覆盖方法指令时，没有其他线程进入该方法区域。可以通过暂停其他所有线程来实现，但是这种方法太重了，这里我在覆盖指令前，将Target方法EntryPoint替换为 art_quick_to_interpreter_bridge ，避免了之后访问需要覆盖的方法区，在覆盖完成后恢复Target方法EntryPoint。 
Inline模式Hook失败
有几种情况下，Inline模式不能Hook：
JIT编译失败
编译的方法指令长度小于JumpTrampoline长度
Native方法
发现上诉三种情况，一律用EntryPoint替换模式 
InterpreterToInterpreter
在8.0之后，如果在Debug编译版本，使用EntrypPoint替换模式会出现Hook失效的情况，方法调用进入InterpreterTointerpreter，不会用到EntryPoint，这里采用YAHFA的方案，Target方法设置kAccNative来规避，只在Debug版本下修改，Release版本不受影响，不修改。 
指令对齐
对于Thumb2指令集， JumpTrampoline是8字节 ，但Thumb有16位和32位两种模式，也就是说JumpTrampoline覆盖掉的指令有可能是不完整的，因此需要做指令判断，复制完整的指令，可能是8字节，也可能是10字节。
指令修复
Inline模式需要处理pc related指令，这里并不进行实际修复，只需要判断覆盖的指令是否包含有pc related指令，如果包含就使用EntryPoint模式。
方法内联
无论哪种模式，方法内联都可以让Hook失效，因此我们需要想方设法禁止内联。先看看什么情况下方法不被内联。
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
//代理方法不内联
  if (method->IsProxyMethod()) {
    return false;
  }
 
  //递归超过限制不内联
  if (CountRecursiveCallsOf(method) > kMaximumNumberOfRecursiveCalls) {
    return false;
  }
 
  const DexFile::CodeItem* code_item = method->GetCodeItem();
  //native方法不内联
  if (code_item == nullptr) {
    return false;
  }
 
  //方法指令大小超过nline_max_code_units不内联
  size_t inline_max_code_units = compiler_driver_->GetCompilerOptions().GetInlineMaxCodeUnits();
  if (code_item->insns_size_in_code_units_ > inline_max_code_units) {
    return false;
  }
 
  //有异常捕获不内联
  if (code_item->tries_size_ != 0) {
    return false;
  }
 
  //设置了kAccCompileDontBother，这里没有返回false，所以并不能阻止内联
  if (!method->IsCompilable()) {
  }
 
  //Verifiy失败不内联
  if (!method->GetDeclaringClass()->IsVerified()) {
    uint16_t class_def_idx = method->GetDeclaringClass()->GetDexClassDefIndex();
    if (Runtime::Current()->UseJitCompilation() ||
        !compiler_driver_->IsMethodVerifiedWithoutFailures(
            method->GetDexMethodIndex(), class_def_idx, *method->GetDexFile())) {
      return false;
    }
  }
  //静态方法或私有方法关联<clinit>不内联
  if (invoke_instruction->IsInvokeStaticOrDirect() &&
      invoke_instruction->AsInvokeStaticOrDirect()->IsStaticWithImplicitClinitCheck()) {
    return false;
  }
看起来就只有改inline_max_code_units比较保险，这里改为0，也就是所有方法都不内联了。这种方式只能阻止JIT内联，对AOT无效。AOT编译的时候会新建立Runtime环境，而我们只能修改当前Runtime环境。OSR也无能为力。 

四、使用FastHook
提供HookInfo
1
2
3
4
5
6
7
8
9
10
private static String[] mHookItem = {
            "mode",
            "targetClassName","targetMethodName","targetParamSig",
            "hookClassName","hookMethodName","hookParamSig",
            "forwardClassName","forwardMethodName","forwardParamSig"
};
 
public static String[][] HOOK_ITEMS = {
             mHookItem
};
注意事项：
HookInfo类可以是任意类，但是必须存在一个名为HOOK_ITEMS的静态二维数组成员变量
HookItem的格式是固定的，如上图所示，mode有两个取值："1"Inline模式；"2"EntryPoint替换模式，特别注意，sig要求的是参数签名而不是完整的方法签名
Hook接口
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
/**
 *
 *@param hookInfoClassName HookInfo类名
 *@param hookInfoClassLoader HookInfo类所在的ClassLoader，如果为null，代表当前ClassLoader
 *@param targetClassLoader Target方法所在的ClassLoader，如果为null，代表当前ClassLoader
 *@param hookClassLoader Hook方法所在的ClassLoader，如果为null，代表当前ClassLoader
 *@param forwardClassLoader Forward方法所在的ClassLoader，如果为null，代表当前ClassLoader
 *@param jitInline 是否内联，false，禁止内联；true，允许内联
 *
 */
public static void doHook(String hookInfoClassName, ClassLoader hookInfoClassLoader, ClassLoader targetClassLoader, ClassLoader hookClassLoader, ClassLoader forwardClassLoader, boolean jitInline)
 
//插件式Hook，建议在attachBaseContext方法里调用
FastHookManger.doHook("hookInfoClassName",pluginsClassloader,null,pluginsClassloader,pluginsClassloader,false);
 
//内置Hook，建议在attachBaseContext方法里调用
FastHookManger.doHook("hookInfoClassName",null,null,null,null,false);
 
//如果能修改framework代码，或者是一些插件框架，在handleBindApplication方法里合适的地方调用，一般在加载apk后，调用attachBaseContext前
FastHookManger.doHook("hookInfoClassName",pluginsClassloader,apkClassLoader,pluginsClassloader,pluginsClassloader,false);
支持Android版本
5.0 - 9.0
支持架构
Thumb2
Arm64

五、参考
YAHFA
Nougat_dlfunctions
