#ClassLoader
By Howie.Hxu

#前言
关于ClassLoader的介绍，可以参考之前提到的：  
[Android动态加载基础 ClassLoader工作机制](https://segmentfault.com/a/1190000004062880)  
另外文章会提到，android中classloader都是采用了“双亲委派机制”，关于这一点可以参考：  
[Parents Delegation Model](http://www.voidcn.com/blog/sun7545526/article/p-1448559.html)

>简单总结一下：  
>对于android中的classloader是按照以下的flow：
>>loadClass方法在加载一个类的实例的时候:    
>>会先查询当前ClassLoader实例是否加载过此类，有就返回；  
>>如果没有。查询Parent是否已经加载过此类，如果已经加载过，就直接返回Parent加载的类；  
>>如果继承路线上的ClassLoader都没有加载，才由Child执行类的加载工作；  

>这样做的好处：
>>首先是共享功能，一些Framework层级的类一旦被顶层的ClassLoader加载过就缓存在内存里面，以后任何地方用到都不需要重新加载。  
>>除此之外还有隔离功能，不同继承路线上的ClassLoader加载的类肯定不是同一个类，这样的限制避免了用户自己的代码冒充核心类库的类访问核心类库包可见成员的情况。这也好理解，一些系统层级的类会在系统初始化的时候被加载，比如java.lang.String，如果在一个应用里面能够简单地用自定义的String类把这个系统的String类给替换掉，那将会有严重的安全问题。

#DexClassLoader和PathClassLoader
关于他们两个恩怨情仇，[Android动态加载基础 ClassLoader工作机制](https://segmentfault.com/a/1190000004062880) 已经有说明。  
>直接说结论：
>>DexClassLoader可以加载jar/apk/dex，可以从SD卡中加载未安装的apk；  
>>PathClassLoader只能加载系统中已经安装过的apk；  

通过源代码可以知道，DexClassLoader和PathClassLoader都是继承自BaseDexClassLoader

```java
	// DexClassLoader.java
	public class DexClassLoader extends BaseDexClassLoader {
	    public DexClassLoader(String dexPath, String optimizedDirectory,
	            String libraryPath, ClassLoader parent) {
	        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
	    }
	}
	
	// PathClassLoader.java
	public class PathClassLoader extends BaseDexClassLoader {
	    public PathClassLoader(String dexPath, ClassLoader parent) {
	        super(dexPath, null, null, parent);
	    }
	    public PathClassLoader(String dexPath, String libraryPath,
	            ClassLoader parent) {
	        super(dexPath, null, libraryPath, parent);
	    }
	}
```

这边以DexClassLoader为例，trace一下整个ClassLoader的初始化过程。  
其中有两个问题比较重要：
1. dex文件如何被load进来，产生了class文件
2. dex文件与oat文件的转化

#DexClassLoader

```java
	super(dexPath, new File(optimizedDirectory), libraryPath, parent);
```

这里的四个参数作用如下：  
1. dexPath：dex文件的路径  
2. new File(optimizedDirectory)：解析出来的dex文件存放的路径  
3. libraryPath：native library的存放路径  
4. parent：父classloader  

#BaseDexClassLoader
继续看super类BaseDexClassLoader：

```java
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
```

其中super（parnent)会call到ClassLoader的构造函数：

```java
    protected ClassLoader(ClassLoader parentLoader) {
        this(parentLoader, false);
    }

    ClassLoader(ClassLoader parentLoader, boolean nullAllowed) {
        if (parentLoader == null && !nullAllowed) {
            throw new NullPointerException("parentLoader == null && !nullAllowed");
        }
        parent = parentLoader;
    }
```

这边只是简单赋了一下parent成员变量。

##DexPathList

```java
    public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {
        ......
        this.dexElements = makePathElements(splitDexPath(dexPath), optimizedDirectory,
                                            suppressedExceptions);
        ......
    }
```

这边的传入参数可以看到，会先把dexPath做切割继续丢下去

##DexPathList.makePathElements
在makePathElements中主要是构造dexElements对象，具体的依据就是传入的files list，即dexPath。

```java
    private static Element[] makePathElements(List<File> files, File optimizedDirectory,
                                              List<IOException> suppressedExceptions) {
        List<Element> elements = new ArrayList<>();
        for (File file : files) {
            ......
            dex = loadDexFile(file, optimizedDirectory);
			......
            elements.add(new Element(dir, false, zip, dex));
			......
        }
        return elements.toArray(new Element[elements.size()]);
    }
```

##DexPathList.loadDexFile
观察这个函数的入参，file对应的是dex路径，optimizedDirectory则是对应的释放dex文件地址。  
简单来说，**src：file，dst：optimizedDirectory**

```java
    private static DexFile loadDexFile(File file, File optimizedDirectory)
            throws IOException {
        if (optimizedDirectory == null) {
            return new DexFile(file);
        } else {
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0);
        }
    }
```

这边遇到了一个optimizedPath，看看他是怎么来的（其实就是一个路径转换罢了）：

```java
    private static String optimizedPathFor(File path,
            File optimizedDirectory) {
        ......
        File result = new File(optimizedDirectory, fileName);
        return result.getPath();
    }
```

##DexFile.loadDex
静态方法，并没有什么特别的东西，其中会new DexFile：

```java
    static public DexFile loadDex(String sourcePathName, String outputPathName,
        int flags) throws IOException {
        return new DexFile(sourcePathName, outputPathName, flags);
    }
```

##DexFile Construct
这里在刚开始的时候回做一个权限检测，如果当前的文件目录所有者跟当前进程的不一致，那就GG。  
当然了，这一句不是重点，重点仍旧在后面：openDexFile

```java
    private DexFile(String sourceName, String outputName, int flags) throws IOException {
        if (outputName != null) {
            try {
                String parent = new File(outputName).getParent();
                if (Libcore.os.getuid() != Libcore.os.stat(parent).st_uid) {
                    ...
                }
            } catch (ErrnoException ignored) {
                // assume we'll fail with a more contextual error later
            }
        }

        mCookie = openDexFile(sourceName, outputName, flags);
        mFileName = sourceName;
        guard.open("close")
    }
```

##DexFile.openDexFile

```java
    private static Object openDexFile(String sourceName, String outputName, int flags) throws IOException {
        // Use absolute paths to enable the use of relative paths when testing on host.
        return openDexFileNative(new File(sourceName).getAbsolutePath(),
                                 (outputName == null) ? null : new File(outputName).getAbsolutePath(),
                                 flags);
    }
```

code trace到这边，基本上java层就已经到底了。因为openDexFileNative会直接call到jni层了：

```java
	private static native Object openDexFileNative(String sourceName, String outputName, int flags);
```

再次来回顾一下这边的三个参数：

1. String sourceName：dex文件的所在地址
2. String outputName：释放dex文件的目标地址
3. int flags：0

#JNI Native
##Register JNI Method
DexFile对应的jni函数在：art/runtime/native/dalvik_system_DexFile.cc
>grep大法好！

```java
	static JNINativeMethod gMethods[] = {
	  ......
	  NATIVE_METHOD(DexFile, openDexFileNative,
	                "(Ljava/lang/String;Ljava/lang/String;I)Ljava/lang/Object;"),
	};
	
	void register_dalvik_system_DexFile(JNIEnv* env) {
	  REGISTER_NATIVE_METHODS("dalvik/system/DexFile");
	}
```

###Macro的定义
可以看到，在这里有两个Macro，他们究竟是什么呢？
>庐山真面目在：art/runtime/jni_internal.h

```c++
	#ifndef NATIVE_METHOD
	#define NATIVE_METHOD(className, functionName, signature) \
	  { #functionName, signature, reinterpret_cast<void*>(className ## _ ## functionName) }
	#endif
	#define REGISTER_NATIVE_METHODS(jni_class_name) \
	  RegisterNativeMethods(env, jni_class_name, gMethods, arraysize(gMethods))
```

###JNINativeMethod

```java
	typedef struct {
	    const char* name;
	    const char* signature;
	    void*       fnPtr;
	} JNINativeMethod;
```

###Macro合体

```c++	
	NATIVE_METHOD(DexFile, openDexFileNative,
	                "(Ljava/lang/String;Ljava/lang/String;I)Ljava/lang/Object;"),
```

展开后

```c++
	"openDexFileNative",
	"(Ljava/lang/String;Ljava/lang/String;I)Ljava/lang/Object;",
	reinterpret_cast<void*> DexFile_openDexFileNative,
```

很自然，就是对应了JNINativeMethod中的name，signature以及fnPtr。  
因此对应native层的function ： **DexFile_openDexFileNative** 

##DexFile_openDexFileNative
native层的代码用到了很多C++ STL库的相关知识，另外还有不少namespace的相关概念。  
STL：[C++:STL标准入门汇总](http://www.cnblogs.com/shiyangxt/archive/2008/09/11/1289493.html)，[三十分钟掌握STL ](http://net.pku.edu.cn/~yhf/UsingSTL.htm)   
namespace：[详解C++中命名空间的意义和用法](http://www.jizhuomi.com/software/289.html)

```java		
	static jobject DexFile_openDexFileNative(
    JNIEnv* env, jclass, jstring javaSourceName, jstring javaOutputName, jint) {
	  ......
	  ClassLinker* linker = Runtime::Current()->GetClassLinker();
	  ......
	  dex_files = linker->OpenDexFilesFromOat(sourceName.c_str(), outputName.c_str(), &error_msgs);
		......
	}
```

ClassLinker是一个很重要的东西，它与art息息相关。简单来说它的创建是伴随着runtime init的时候起来的，可以认为是每个art &　进程会维护一个对象（具体代码还没有分析，只是从网上看了一篇[文章](http://www.iloveandroid.net/2015/12/20/AndroidART-8/)）  
这里我们暂时放下ClassLinker的创建初始化职责权限，先看看OpenDexFilesFromOat

##ClassLinker::OpenDexFilesFromOat
这个函数非常大，其主要的几个动作如下：  
1. 构造OatFileAssistant对象：这个对象的主要作用是协助dex文件解析成oat文件  
2. 遍历ClassLinker的成员变量：oat\_files\_，目的是查看当前的dex文件是否已经被解析成oat了  
 1. 如果当前dex已经被解析成了oat文件，那么就跳过dex的解析，直接进入3）   
 2. 反之，如果当前dex文件并没有被解析过，那么就会走：**oat\_file\_assistant.MakeUpToDate**    
3. **填充dex\_files**对象并返回，这里也有两种情况（区分oat or dex）：  
 1.	oat\_file\_assistant.LoadDexFiles
 2.	DexFile::Open

其中我们需要关注的点其实只有两点（上文划出的重点）共两个函数：  
1. oat\_file\_assistant.MakeUpToDate  
2. oat\_file\_assistant.LoadDexFiles  

```c++
	std::vector<std::unique_ptr<const DexFile>> ClassLinker::OpenDexFilesFromOat(
	    const char* dex_location, const char* oat_location,
	    std::vector<std::string>* error_msgs) {
	  ......
	  OatFileAssistant oat_file_assistant(dex_location, oat_location, kRuntimeISA,
	     !Runtime::Current()->IsAotCompiler());
	  ......
	  const OatFile* source_oat_file = nullptr;
	  ......
	    for (const OatFile* oat_file : oat_files_) {
	      CHECK(oat_file != nullptr);
	      if (oat_file_assistant.GivenOatFileIsUpToDate(*oat_file)) {
	        source_oat_file = oat_file;
	        break;
	      }
	    }
	  ......
	
	  if (source_oat_file == nullptr) {
	    if (!oat_file_assistant.MakeUpToDate(&error_msg)) {
	      ......
	    }
	
	    // Get the oat file on disk.
	    std::unique_ptr<OatFile> oat_file = oat_file_assistant.GetBestOatFile();
	    if (oat_file.get() != nullptr) {
	      	......
	        if (!DexFile::MaybeDex(dex_location)) {
	          accept_oat_file = true;
	          ......
	        }
	      }
	
	      if (accept_oat_file) {
	        source_oat_file = oat_file.release();
	        RegisterOatFile(source_oat_file);
	      }
	    }
	  }
	
	  std::vector<std::unique_ptr<const DexFile>> dex_files;
	  ......
	  if (source_oat_file != nullptr) {
	    dex_files = oat_file_assistant.LoadDexFiles(*source_oat_file, dex_location);
	    ......
	  }
	
	  // Fall back to running out of the original dex file if we couldn't load any
	  // dex_files from the oat file.
	  if (dex_files.empty()) {
	    ......
	        if (!DexFile::Open(dex_location, dex_location, &error_msg, &dex_files)) {
	    ......
	  }
	  return dex_files;
	}
```

下面的内容会牵扯到很多oat file的解析，这一块的内容有点庞大，我也理解的不是很透彻。  
如果有兴趣的话可以先阅读一下[《罗升阳：Android运行时ART加载OAT文件的过程分析》](http://blog.csdn.net/luoshengyang/article/details/39307813)，其中对于oat file format会有进一步的了解

这一部分的知识对于理顺classloader的flow来说障碍不大，但是对于了解整个art的flow来说至关重要。

###OatFileAssistant::MakeUpToDate
一出4岔口的switch分支，对于这边我们关注的是kDex2OatNeeded（可以认为是第一次加载，这样就省略掉很多细节）  
另外千万不要小看**GetDexOptNeeded()**，在它内部会去读取dex文件，检查checksum

```c++
	bool OatFileAssistant::MakeUpToDate(std::string* error_msg) {
	  switch (GetDexOptNeeded()) {
	    case kNoDexOptNeeded: return true;
	    case kDex2OatNeeded: return GenerateOatFile(error_msg);
	    case kPatchOatNeeded: return RelocateOatFile(OdexFileName(), error_msg);
	    case kSelfPatchOatNeeded: return RelocateOatFile(OatFileName(), error_msg);
	  }
	  UNREACHABLE();
	}
```

由于我们这里是第一次load，所以就会进入到GenerateOatFile

####OatFileAssistant::GenerateOatFile
可以看到这支函数的主要最用是call Dex2Oat，在最终call之前会去检查一下文件是否存在。  

```c++
	bool OatFileAssistant::GenerateOatFile(std::string* error_msg) {
	  ......
	  std::vector<std::string> args;
	  args.push_back("--dex-file=" + std::string(dex_location_));
	  args.push_back("--oat-file=" + oat_file_name);
	  ......
	  if (!OS::FileExists(dex_location_)) {
	    ......
	    return false;
	  }
	  if (!Dex2Oat(args, error_msg)) {
	    ......
	  }
	  ......
	  return true;
	}
```

####OatFileAssistant::Dex2Oat
argv是此处的关键，不光会传入dst，src等基本信息，还会有诸如debugger，classpath等信息

```c++
	bool OatFileAssistant::Dex2Oat(const std::vector<std::string>& args,
	                               std::string* error_msg) {
	  ......
	  std::vector<std::string> argv;
	  argv.push_back(runtime->GetCompilerExecutable());
	  argv.push_back("--runtime-arg");
	  ......
	  return Exec(argv, error_msg);
	}
```

至于文末的Exec，则会来到art/runtime/utils.cc  
bool Exec(std::vector<std::string>& arg_vector, std::string* error_msg)

####Exec
这个函数的重点是在fork，而fork之后会在子进程执行arg带入的第一个参数指定的那个可执行文件。  
而在fork的父进程，则会继续监听子进程的运行状态并一直wait，所以对于上层来说..这边就是一个同步调用了。

```c++
	bool Exec(std::vector<std::string>& arg_vector, std::string* error_msg) {
	  ......
	  const char* program = arg_vector[0].c_str();
	  ......
	  pid_t pid = fork();
	  if (pid == 0) {
	    ......
	    execv(program, &args[0]);
		......
	  } else {
	    ......
	    pid_t got_pid = TEMP_FAILURE_RETRY(waitpid(pid, &status, 0));
	    ......
	  return true;
	}
```

再回头看一下argv的第一个参数：**argv.push_back(runtime->GetCompilerExecutable());**  
所以..如果非debug状态下，我们用到可执行文件就是：**/bin/dex2oat**

```c++
	std::string Runtime::GetCompilerExecutable() const {
	  if (!compiler_executable_.empty()) {
	    return compiler_executable_;
	  }
	  std::string compiler_executable(GetAndroidRoot());
	  compiler_executable += (kIsDebugBuild ? "/bin/dex2oatd" : "/bin/dex2oat");
	  return compiler_executable;
	}
```

所以到这边，我们大致就了解了dex文件是如何转换到oat文件的，所以也就是回答了最早提出的问题2。  
从Exec返回之后，我们还会继续去检查一下oat文件是否真的弄好了，如果一切正常，最后就会通过

```c++
	RegisterOatFile(source_oat_file);
```

把对应的oat file加入到ClassLinker的成员变量：oat\_files\_，下次就不用继续执行dex2oat了。

###OatFileAssistant::LoadDexFiles
对于这一段的flow我之前一直很不解，为什么之前要把dex转换成oat，这边看起来又要通过oat去获取dex？
其实简单来想这里应该是两种情况：  
1. 首先是通过dex拿到oat文件  
2. 构造dex structure给上层  
所以光从字面意义来解释这个函数，就会让人进入一种误区，我们来看code：

```c++
	std::vector<std::unique_ptr<const DexFile>> OatFileAssistant::LoadDexFiles(
	    const OatFile& oat_file, const char* dex_location) {
	  std::vector<std::unique_ptr<const DexFile>> dex_files;
	
	  // Load the primary dex file.
	  ......
	  const OatFile::OatDexFile* oat_dex_file = oat_file.GetOatDexFile(
	      dex_location, nullptr, false);
	  ......
	  std::unique_ptr<const DexFile> dex_file = oat_dex_file->OpenDexFile(&error_msg);
	  ......
	  dex_files.push_back(std::move(dex_file));
	
	  // Load secondary multidex files
	  for (size_t i = 1; ; i++) {
	    std::string secondary_dex_location = DexFile::GetMultiDexLocation(i, dex_location);
	    oat_dex_file = oat_file.GetOatDexFile(secondary_dex_location.c_str(), nullptr, false);
	    ......
	    dex_file = oat_dex_file->OpenDexFile(&error_msg);
	    ......
	  }
	  return dex_files;
	}
```

这里的实现看起来比较复杂，但其实就是：  
1. 分别load primary dex和secondary multidex
2. 再通过OpenDexFile方法获得DexFile对象。
>因为对于oat文件来说，它可以是由多个dex产生的，详细的情况可以[参考老罗的文章](http://blog.csdn.net/luoshengyang/article/details/39307813)，比如boot.oat。

所以，我们需要看看这边OpenDexFile到底做了什么：

```c++
	std::unique_ptr<const DexFile> OatFile::OatDexFile::OpenDexFile(std::string* error_msg) const {
	  return DexFile::Open(dex_file_pointer_, FileSize(), dex_file_location_,
	                       dex_file_location_checksum_, this, error_msg);
	}
```

其中**dex_file_pointer_**是oat文件中dex数据的起始地址（DexFile Meta段）

###DexFile::Open
上文提到的Open原型是：

```c++
	  static std::unique_ptr<const DexFile> Open(const uint8_t* base, size_t size,
	                                             const std::string& location,
	                                             uint32_t location_checksum,
	                                             const OatDexFile* oat_dex_file,
	                                             std::string* error_msg) {
	    return OpenMemory(base, size, location, location_checksum, nullptr, oat_dex_file, error_msg);
	  }
```

继续trace一下：

```c++
	std::unique_ptr<const DexFile> DexFile::OpenMemory(const uint8_t* base,
	                                                   size_t size,
	                                                   const std::string& location,
	                                                   uint32_t location_checksum,
	                                                   MemMap* mem_map,
	                                                   const OatDexFile* oat_dex_file,
	                                                   std::string* error_msg) {
	  CHECK_ALIGNED(base, 4);  // various dex file structures must be word aligned
	  std::unique_ptr<DexFile> dex_file(
	      new DexFile(base, size, location, location_checksum, mem_map, oat_dex_file));
	  if (!dex_file->Init(error_msg)) {
	    dex_file.reset();
	  }
	  return std::unique_ptr<const DexFile>(dex_file.release());
	}
```

最后看一下构造函数：

```c++
	DexFile::DexFile(const uint8_t* base, size_t size,
	                 const std::string& location,
	                 uint32_t location_checksum,
	                 MemMap* mem_map,
	                 const OatDexFile* oat_dex_file)
	    : begin_(base),
	      size_(size),
	      location_(location),
	      location_checksum_(location_checksum),
	      mem_map_(mem_map),
	      header_(reinterpret_cast<const Header*>(base)),
	      string_ids_(reinterpret_cast<const StringId*>(base + header_->string_ids_off_)),
	      type_ids_(reinterpret_cast<const TypeId*>(base + header_->type_ids_off_)),
	      field_ids_(reinterpret_cast<const FieldId*>(base + header_->field_ids_off_)),
	      method_ids_(reinterpret_cast<const MethodId*>(base + header_->method_ids_off_)),
	      proto_ids_(reinterpret_cast<const ProtoId*>(base + header_->proto_ids_off_)),
	      class_defs_(reinterpret_cast<const ClassDef*>(base + header_->class_defs_off_)),
	      find_class_def_misses_(0),
	      class_def_index_(nullptr),
	      oat_dex_file_(oat_dex_file) {
	  CHECK(begin_ != nullptr) << GetLocation();
	  CHECK_GT(size_, 0U) << GetLocation();
	}
```

其实就是一些数据的填充，然后再打包传回到上层（java端），丢该DexFile的mCookie对象。
：）