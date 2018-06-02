title: AndFix源码解析
date: 2018-05-31 16:47:33
category: 源码分析
tags: [热修复,源码]
---
#### 简介
AndFix是阿里开源的热修复框架。其支持android2.3-7.0版本。可在不重启应用的情况下hook方法。主要用于紧急修复bug。

优点：

- 支持ART与Dalvik
- 签名验证 安全性高
- 轻便简单
- 无需重启

缺点：
-  不支持新增成员变量和修改成员变量
-  不支持新增类
-  不支持修改资源文件
-  由于是在native层替换方法，某些手机修改过底层实现，可能会出现兼容问题

#### 原理
官方提供的apkpatch.jar工具，通过smali/baksmal开源库比对两个apk中dex的不同，将其反汇编为smali文件后在改动的方法上添加注解，然后把smali文件汇编为dex，签名并打包为补丁文件。应用下载补丁后，校验签名，加载其中的dex类，根据添加的注解获取到需要改动的方法以及原始类方法信息，然后将其传到native层修改原始方法的结构体相关指针到新方法上，以达到hook的目的。

#### 生成修复补丁
生成差异补丁主要是依赖apkpatch.jar这个工具，反编译查看代码。它主要依赖了smali/baksmal开源库进行dex的比对和生成。

首先从eular.patch包下Main类中的main方法开始：

```java
public class Main
{
  public static void main(String[] args)
  {
    //命令行解析工具类
    CommandLineParser parser = new PosixParser();
    CommandLine commandLine = null;
    option();
    ...
    //解析传入的参数
    commandLine = parser.parse(allOptions, args);
    ...
    //创建ApkPatch对象，并传入参数
    ApkPatch apkPatch = new ApkPatch(from, to, name, out, keystore, 
        password, alias, entry);
    //执行patch
    apkPatch.doPatch();
  }
```
看下ApkPatch的doPatch方法。生成patch文件主要有四步：第一步对比dex的不同；第二步反汇编dex到smali并生成新的dex；第三步签名；第四步md5加密；
```java
 public void doPatch()
  {
      ...
      //用于存储smali的文件夹
      File smaliDir = new File(this.out, "smali");
      if (!smaliDir.exists()) {
        smaliDir.mkdir();
      }
      ...
      //1.对比两个apk中dex的不同，并把信息存储在该类中
      DiffInfo info = new DexDiffer().diff(this.from, this.to);
      //2.反汇编为smali，并添加注解信息后汇编为dex文件
      this.classes = buildCode(smaliDir, dexFile, info);
      //3.签名文件并写入打包配置
      build(outFile, dexFile);
      //4.计算md5并重命名
      release(this.out, dexFile, outFile);
      ...
  }
```

##### 第一步，对比dex的不同。

在DexDiffer的diff方法中：

```java
public DiffInfo diff(File newFile, File oldFile)
    throws IOException
  {
    //新的dex文件
    DexBackedDexFile newDexFile = DexFileFactory.loadDexFile(newFile, 19, 
      true);
    //旧的dex文件
    DexBackedDexFile oldDexFile = DexFileFactory.loadDexFile(oldFile, 19, 
      true);
    DiffInfo info = DiffInfo.getInstance();
    boolean contains = false;
    for (DexBackedClassDef newClazz : newDexFile.getClasses())
    {
      Set<? extends DexBackedClassDef> oldclasses = oldDexFile
        .getClasses();
      //逐个比对新旧dex
      for (DexBackedClassDef oldClazz : oldclasses) {
        if (newClazz.equals(oldClazz))
        {
          //比对field
          compareField(newClazz, oldClazz, info);
          //比对method
          compareMethod(newClazz, oldClazz, info);
          contains = true;
          break;
        }
      }
      //添加新加的类（其实重新打包的dex差异文件中并不包含）
      if (!contains) {
        info.addAddedClasses(newClazz);
      }
    }
    return info;
  }
```
比较field的初始值的异同：
```java
public void compareField(DexBackedField object, Iterable<? extends DexBackedField> olds, DiffInfo info)
  {
    for (DexBackedField reference : olds) {
      if (reference.equals(object))
      {
        if ((reference.getInitialValue() == null) && 
          (object.getInitialValue() != null))
        {
          info.addModifiedFields(object);
          return;
        }
        if ((reference.getInitialValue() != null) && 
          (object.getInitialValue() == null))
        {
          info.addModifiedFields(object);
          return;
        }
        if ((reference.getInitialValue() == null) && 
          (object.getInitialValue() == null)) {
          return;
        }
        if (reference.getInitialValue().compareTo(
          object.getInitialValue()) != 0)
        {
          info.addModifiedFields(object);
          return;
        }
        return;
      }
    }
    info.addAddedFields(object);
  }
```
比较method实现的异同：
```java
public void compareMethod(DexBackedMethod object, Iterable<? extends DexBackedMethod> olds, DiffInfo info)
  {
    for (DexBackedMethod reference : olds) {
      if (reference.equals(object))
      {
        if ((reference.getImplementation() == null) && 
          (object.getImplementation() != null))
        {
          info.addModifiedMethods(object);
          return;
        }
        if ((reference.getImplementation() != null) && 
          (object.getImplementation() == null))
        {
          info.addModifiedMethods(object);
          return;
        }
        if ((reference.getImplementation() == null) && 
          (object.getImplementation() == null)) {
          return;
        }
        if (!reference.getImplementation().equals(object.getImplementation()))
        {
          info.addModifiedMethods(object);
          return;
        }
        return;
      }
    }
    info.addAddedMethods(object);
  }
```
##### 第二步，反汇编dex并生成新的dex。

这里只简单介绍下buildcode方法的代码，详细实现细节可以深入了解下smali/baksmal开源库。代码如下：

```java
private static Set<String> buildCode(File smaliDir, File dexFile, DiffInfo info)
    throws IOException, RecognitionException, FileNotFoundException
  {
    Set<String> classes = new HashSet();
    //保存所有改动class的dex信息
    Set<DexBackedClassDef> list = new HashSet();
    list.addAll(info.getAddedClasses());
    list.addAll(info.getModifiedClasses());
    
    //设置反汇编dex的参数
    baksmaliOptions options = new baksmaliOptions();
    ...
    //反汇编dex到这里
    ClassFileNameHandler outFileNameHandler = new ClassFileNameHandler(
      smaliDir, ".smali");
    //修改smali后从这里在汇编为dex
    ClassFileNameHandler inFileNameHandler = new ClassFileNameHandler(
      smaliDir, ".smali");
    DexBuilder dexBuilder = DexBuilder.makeDexBuilder();
    for (DexBackedClassDef classDef : list)
    {
      String className = classDef.getType();
      //把dex反汇编到outFile中
      baksmali.disassembleClass(classDef, outFileNameHandler, options);
      File smaliFile = inFileNameHandler.getUniqueFilenameForClass(
        TypeGenUtil.newType(className));
      classes.add(TypeGenUtil.newType(className)
        .substring(1, TypeGenUtil.newType(className).length() - 1)
        .replace('/', '.'));
      //读取smali添加注解并汇编到outFile中
      SmaliMod.assembleSmaliFile(smaliFile, dexBuilder, true, true);
    }
    //把修改后的smile写入到dexfile中
    dexBuilder.writeTo(new FileDataStore(dexFile));
    return classes;
  }
```
##### 第三步，对修改后的dex签名，并写入配置文件。

其中签名和校验原理可以参考[这里](https://blog.csdn.net/hp910315/article/details/77684725)。

```java
protected void build(File outFile, File dexFile)
    throws KeyStoreException, FileNotFoundException, IOException, NoSuchAlgorithmException, CertificateException, UnrecoverableEntryException
  {
    KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
    KeyStore.PrivateKeyEntry privateKeyEntry = null;
    InputStream is = new FileInputStream(this.keystore);
    keyStore.load(is, this.password.toCharArray());
    //获取keystore的privateKeyEntry
    privateKeyEntry = (KeyStore.PrivateKeyEntry)keyStore.getEntry(this.alias, 
      new KeyStore.PasswordProtection(this.entry.toCharArray()));
    //签名
    PatchBuilder builder = new PatchBuilder(outFile, dexFile, 
      privateKeyEntry, System.out);
    //写入META-INF/PATCH.MF配置文件
    builder.writeMeta(getMeta());
    builder.sealPatch();
  }
```
##### 第四步，计算文件md5并重命名文件。

```java
protected void release(File outDir, File dexFile, File outFile)
    throws NoSuchAlgorithmException, FileNotFoundException, IOException
  {
    MessageDigest messageDigest = MessageDigest.getInstance("md5");
    FileInputStream fileInputStream = new FileInputStream(dexFile);
    byte[] buffer = new byte[' '];
    int len = 0;
    while ((len = fileInputStream.read(buffer)) > 0) {
      messageDigest.update(buffer, 0, len);
    }
    String md5 = HexUtil.hex(messageDigest.digest());
    fileInputStream.close();
    outFile.renameTo(new File(outDir, this.name + "-" + md5 + ".apatch"));
  }
```
#### 加载与替换

客户端下载patch文件后，首先获取打包其中的配置信息。然后逐个遍历dex文件，如果是需要fix的类，则获取其中MethodReplace注解信息，注解信息包含了需要hook的原始方法以及类信息。把待替换的类方法信息以及原始的类方法信息传入到native层，修改原始方法的结构体指针到达到hook的目的。

加载patch文件，并获取打包配置信息的代码如下：
```java
public void addPatch(String path) throws IOException {
        //patch文件
		...
		FileUtil.copyFile(src, dest);// copy to patch's directory
		//1.添加patch并解析配置信息
		Patch patch = addPatch(dest);
		if (patch != null) {
			//2.替换patch
			loadPatch(patch);
		}
	}
```
看下addPatch如何添加并解析配置信息的：
```java
private Patch addPatch(File file) {
		Patch patch = null;
		if (file.getName().endsWith(SUFFIX)) {
			try {
			    //创建patch对象，解析配置信息
				patch = new Patch(file);
				mPatchs.add(patch);
			} catch (IOException e) {
				Log.e(TAG, "addPatch", e);
			}
		}
		return patch;
	}
```
在patch对象初始化时会调用init方法，解析构建patch信息
```java
private void init() throws IOException {
		JarFile jarFile = null;
		InputStream inputStream = null;
		try {
			jarFile = new JarFile(mFile);
            //获取META-INF配置文件的配置信息
			JarEntry entry = jarFile.getJarEntry(ENTRY_NAME);
			inputStream = jarFile.getInputStream(entry);
			Manifest manifest = new Manifest(inputStream);
			Attributes main = manifest.getMainAttributes();
            //获取META-INF中的Patch-Name值
			mName = main.getValue(PATCH_NAME);
			mTime = new Date(main.getValue(CREATED_TIME));

			mClassesMap = new HashMap<String, List<String>>();
			Attributes.Name attrName;
			String name;
			List<String> strings;
            //遍历META-INF配置信息
			for (Iterator<?> it = main.keySet().iterator(); it.hasNext();) {
				attrName = (Attributes.Name) it.next();
				name = attrName.toString();
				if (name.endsWith(CLASSES)) {
					strings = Arrays.asList(main.getValue(attrName).split(","));
                    //添加Patch-Classes的fix的类信息
					if (name.equalsIgnoreCase(PATCH_CLASSES)) {
						mClassesMap.put(mName, strings);
					} else {
						mClassesMap.put(
								name.trim().substring(0, name.length() - 8),// remove
																			// "-Classes"
								strings);
					}
				}
			}
		} 
        ...
	}
```
解析完配置信息后，加载patch文件
```java
private void loadPatch(Patch patch) {
		Set<String> patchNames = patch.getPatchNames();
		ClassLoader cl;
		List<String> classes;
		for (String patchName : patchNames) {
            //获取classloader
			if (mLoaders.containsKey("*")) {
				cl = mContext.getClassLoader();
			} else {
				cl = mLoaders.get(patchName);
			}
			if (cl != null) {
                //根据patchName获取fix的class字符串信息
				classes = patch.getClasses(patchName);
                //fix方法
				mAndFixManager.fix(patch.getFile(), cl, classes);
			}
		}
	}
```
看下fix的方法
```java
public synchronized void fix(File file, ClassLoader classLoader,
			List<String> classes) {
		    ...
		    //加载dexFile
			final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
					optfile.getAbsolutePath(), Context.MODE_PRIVATE);

			if (saveFingerprint) {
				mSecurityChecker.saveOptSig(optfile);
			}
		    //创建classLoader
			...
			Enumeration<String> entrys = dexFile.entries();
			Class<?> clazz = null;
			//逐个遍历每个dex文件
			while (entrys.hasMoreElements()) {
				String entry = entrys.nextElement();
				//如果没有在配置类中找到该类或者是空则跳过
				if (classes != null && !classes.contains(entry)) {
					continue;// skip, not need fix
				}
				//从dex中加载fix的class
				clazz = dexFile.loadClass(entry, patchClassLoader);
				if (clazz != null) {
				    //调用fixClass去hook
					fixClass(clazz, classLoader);
				}
			}
		} catch (IOException e) {
			Log.e(TAG, "pacth", e);
		}
	}
```
看下fixclass的方法
```java
private void fixClass(Class<?> clazz, ClassLoader classLoader) {
		//获取fix的class类的所有方法
		Method[] methods = clazz.getDeclaredMethods();
		MethodReplace methodReplace;
		String clz;
		String meth;
		//逐个遍历方法获取带有MethodReplace注解的方法
		for (Method method : methods) {
			methodReplace = method.getAnnotation(MethodReplace.class);
			if (methodReplace == null)
				continue;
			//需要替换方法的原始类
			clz = methodReplace.clazz();
			//需要替换原始类中的方法
			meth = methodReplace.method();
			if (!isEmpty(clz) && !isEmpty(meth)) {
				//进行方法的替换
				replaceMethod(classLoader, clz, meth, method);
			}
		}
	}
```
replaceMethod方法会判断原来的类有没有加载，没有加载会先加载，然后调用native去hook
```java
private void replaceMethod(ClassLoader classLoader, String clz,
			String meth, Method method) {
		try {
			String key = clz + "@" + classLoader.toString();
			Class<?> clazz = mFixedClass.get(key);
			//如果原来的类没有加载
			if (clazz == null) {// class not load
				Class<?> clzz = classLoader.loadClass(clz);
				// initialize target class
				clazz = AndFix.initTargetClass(clzz);
			}
			if (clazz != null) {// initialize class OK
				mFixedClass.put(key, clazz);
				Method src = clazz.getDeclaredMethod(meth,
						method.getParameterTypes());
				//调用native层hook方法
				AndFix.addReplaceMethod(src, method);
			}
		} catch (Exception e) {
			Log.e(TAG, "replaceMethod", e);
		}
	}
```
native方法，一个是调用art的一个是调用dalvik的，这里只看下调用dalvik的代码
```java
static void replaceMethod(JNIEnv* env, jclass clazz, jobject src,
		jobject dest) {
	if (isArt) {
		//art的替换方法
		art_replaceMethod(env, src, dest);
	} else {
		//dalvik的替换方法
		dalvik_replaceMethod(env, src, dest);
	}
}

extern void __attribute__ ((visibility ("hidden"))) dalvik_replaceMethod(
		JNIEnv* env, jobject src, jobject dest) {
	//获取对象
	jobject clazz = env->CallObjectMethod(dest, jClassMethod);
	ClassObject* clz = (ClassObject*) dvmDecodeIndirectRef_fnPtr(
			dvmThreadSelf_fnPtr(), clazz);
	//设置类已初始化
	clz->status = CLASS_INITIALIZED;

	Method* meth = (Method*) env->FromReflectedMethod(src);
	Method* target = (Method*) env->FromReflectedMethod(dest);
	LOGD("dalvikMethod: %s", meth->name);
    
    //修改方法结构体信息
//	meth->clazz = target->clazz;
	meth->accessFlags |= ACC_PUBLIC;
	meth->methodIndex = target->methodIndex;
	meth->jniArgInfo = target->jniArgInfo;
	meth->registersSize = target->registersSize;
	meth->outsSize = target->outsSize;
	meth->insSize = target->insSize;

	meth->prototype = target->prototype;
	meth->insns = target->insns;
	meth->nativeFunc = target->nativeFunc;
}
```

#### 补丁验证以及管理
客户端获取到补丁后，不仅仅加载能用就行，还需要考虑不同版本的管理以及安全性等问题。仔细思考下会面临三个问题：第一，如何对补丁进行安全校验，防止被篡改。第二，如何保证补丁在下次发布版本前一直生效。第三，不同版本的补丁该如何管理。

AndFix的解决方案是：第一，对补丁进行签名。并且在补丁第一次加载后存储md5，以后每次加载之前都先验证md5是否相同以防止下载到本地的补丁被篡改。第二，持久化补丁信息，在应用启动时加载所有补丁保证在发布版本前一直生效。第三，删除其他版本补丁，保证只有当前版本可用。

对补丁进行签名校验，以及md5校验，在AndFixManager的fix方法中

```java
public synchronized void fix(File file, ClassLoader classLoader,
			List<String> classes) {
		...
		//校验签名
		if (!mSecurityChecker.verifyApk(file)) {// security check fail
			return;
		}
		...
			File optfile = new File(mOptDir, file.getName());
			boolean saveFingerprint = true;
			//如果文件存在，校验md5
			if (optfile.exists()) {
				// need to verify fingerprint when the optimize file exist,
				// prevent someone attack on jailbreak device with
				// Vulnerability-Parasyte.
				// btw:exaggerated android Vulnerability-Parasyte
				// http://secauo.com/Exaggerated-Android-Vulnerability-Parasyte.html
				//校验文件的md5
				if (mSecurityChecker.verifyOpt(optfile)) {
					saveFingerprint = false;
				} else if (!optfile.delete()) {
					return;
				}
			}

			final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
					optfile.getAbsolutePath(), Context.MODE_PRIVATE);
			if (saveFingerprint) {
			    //第一次的时候保存md5
				mSecurityChecker.saveOptSig(optfile);
			}
		...
```
应用启动加载补丁，以及版本变动删除补丁的逻辑在PatchManager的init方法
```java
public void init(String appVersion) {
		...
		SharedPreferences sp = mContext.getSharedPreferences(SP_NAME,
				Context.MODE_PRIVATE);
		String ver = sp.getString(SP_VERSION, null);
		//如果保存的版本名称和传入的版本名称不同，清除补丁文件，并且保存新的版本
		if (ver == null || !ver.equalsIgnoreCase(appVersion)) {
			cleanPatch();
			sp.edit().putString(SP_VERSION, appVersion).commit();
		} else {
			//初始化补丁文件，会调动addpatch
			initPatchs();
		}
	}

private Patch addPatch(File file) {
		Patch patch = null;
		if (file.getName().endsWith(SUFFIX)) {
			try {
				patch = new Patch(file);
				mPatchs.add(patch);
			} catch (IOException e) {
				Log.e(TAG, "addPatch", e);
			}
		}
		return patch;
	}
```
初始化补丁文件时会把沙盒文件夹下的补丁保存到mPatchs中，当调用loadPatch方法时逐个加载补丁
```java
public void loadPatch() {
		mLoaders.put("*", mContext.getClassLoader());// wildcard
		Set<String> patchNames;
		List<String> classes;
        //逐个加载补丁
		for (Patch patch : mPatchs) {
			patchNames = patch.getPatchNames();
			for (String patchName : patchNames) {
				classes = patch.getClasses(patchName);
				mAndFixManager.fix(patch.getFile(), mContext.getClassLoader(),
						classes);
			}
		}
	}
```

#### 关键点
1. 签名打包和验证签名（安全性问题可以借鉴）
2. smali差异比对（还可以这样玩）
3. native hook（知识深度的重要性）



