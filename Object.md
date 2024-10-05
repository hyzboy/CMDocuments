# ObjectBaseInfo

    size_t          hash_code;          ///<对象数据类型的hash值
    ObjectManager * object_manager;     ///<对象管理器
    size_t          serial_number;      ///<对象序列号

    SourceCodeLocation source_code_location;    ///<对象创建的源代码位置

其中hash_code使用typeid(Object).hash_code()方式获取

serial_number则为唯一序列号

Object类将ObjectBaseInfo信息放至于类的最前方，目的在于当一个Object无法识别时：可以根据ObjectBaseInfo信息进行最简单的甄别。

在较为极端的情况下，我们可以在每个Object的创建时，向日志处输出该Object的具体hash_code和serial_number。当程序崩溃，或是内存泄露时，我们仅需将这些信息输出，即可得出有那些Object没有释放。
同时因为记录了创建时的源代码地址，也可以很容易找到创建对象的代码。

使用hash_code/serial_number的另一好处是我们可以完全不传递Object指针，而只传递hash_code/serial_number给其它模块，甚至是保存在服务器或文件中。到时再根据hash_code/serial_number来获取对应的Object。

# Object
是所有对象的基类，它包含了ObjectBaseInfo以及Deinitailize()函数的定义

# 具体Object的定义

DebugObject.h
    
    #include<hgl/type/object/Object.h>

    using namespace hgl;

    class DebugObject:public Object
    {
        //这里定义了该类的构造与析构函数以及一些其它信息
        HGL_OBJECT_CLASS_BODY(DebugObject)

    public:

        void Initailize()
        {
            //初始化函数，该函数使用模板+Macro定义。函数是可以有参数的，但我们这里没有写。
            std::cout<<"DebugObject::Initailize("<<GetSerialNumber()<<")"<<std::endl;
        }

        void Deinitailize() override
        {
            std::cout<<"DebugObject::Deinitailize("<<GetSerialNumber()<<")"<<std::endl;
        }
    };

DebugObject.cpp
    
    #include<hgl/type/object/ObjectManager.h>

    //这里定义了缺省的对象管理器来管理DebugObject对象
    HGL_DEFINE_DEFAULT_OBJECT_MANAGER(DebugObject);

# 使用定义的Object

    #include<hgl/type/object/DefaultCreateObject.h>

    int main()
    {
        SafePtr<DebugObject> obj1=HGL_NEW_OBJECT(DebugObject);  //New出新的对象

        HGL_DEFINE_OBJECT(DebugObject,obj2);                    //等于上一行

    //    DebugObject *obj3=new DebugObject();                  //编译不过(构造函数被定义为私有)

        obj1.Release();                                         //释放obj1的引用(如引用归0会被delete)

        //delete obj2;                                          //编译不过,SafePtr<>不能被delete

        if(obj1.IsValid())
        {
            std::cerr<<"[ERROR] obj1 IsValid() error!"<<std::endl;
        }
        else
        {
            std::cout<<"[ OK ] obj1 isn't valid!"<<std::endl;
        }

        const size_t obj2_sn=obj2->GetSerialNumber();

        SafePtr<DebugObject> obj2_indirect=GetObjectBySerial<DebugObject>(obj2_sn);    //直接根据序列号获取对象

        if(!obj2_indirect.IsValid())
        {
            std::cerr<<"[ERROR] obj2_indirect isn't valid!"<<std::endl;                //获取失败(不应该)
        }
        else if(obj2_indirect!=obj2)
        {
            std::cerr<<"[ERROR] obj2_indirect!=obj2"<<std::endl;                       //获取的对象不是obj2(不应该)
        }
        else
        {
            std::cout<<"[ OK ] obj2_indirect==obj2"<<std::endl;                        //获取的对象是obj2(成功)

            obj2_indirect.Destory();                                                   //强制销毁obj2_indirect,不管引用是否归0

            if(obj2.IsValid())
            {
                std::cerr<<"[ERROR] obj2 IsValid() error!"<<std::endl;                 //错误，由于obj2_indirect被Destory了,所以obj2也无效才对
            }
            else
            {
                std::cout<<"[ OK ] obj2 isn't valid!"<<std::endl;                      //成功，obj2也跟随无效了
            }
        }

        return 0;

        //obj1会因为SafePtr的定义被自动释放，所以不会造成内存泄露
    }