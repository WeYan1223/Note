#### Binder

> 为什么需要多进程？

* 突破内存限制：`Android` 系统会限制每个进程的最大内存（一般为 16M），引入多进程可以让程序的某些模块运行在另外的进程中，以获取更多的内存
* 风险隔离：对于不稳定的功能模块放到子进程运行，子进程崩溃不会影响主进程



> `Android` 何如开启多进程？

`Android` 中使用多进程的方式只有一种，给四大组件在 `AndroidMenifest` 中指定不同的 `android:process` 属性



> 进程隔离是什么？

* 进程隔离是为保护在操作系统之上运行的进程互不干扰而设计的技术，为了避免进程 A 修改进程 B 数据的情况发生。 

* 进程隔离的实现，使用了虚拟地址空间。对于进程来说，直接接触到的是虚拟地址，至于虚拟地址怎么映射到物理地址，这部分工作由操作系统来负责，这样使得各个进程使用的内存空间的相对独立的。



> 用户空间？内核空间？

* 内核是操作系统的核心，可以访问受保护的内存空间，也具有访问硬件设备的权限
* 内核所在的内存空间称为内核空间，应用程序所在内存空间则称为用户空间
* 应用程序访问内核的资源，需要通过内核提供的系统调用，这样可以保证资源访问都是在内核的控制下执行，以免应用程序程序对系统资源的越权访问，从而保障了系统的安全和稳定
* 当进程执行系统调用而陷入内核代码中执行时，就称进程处于内核运行态，此时处理器处于特权级最高的内核代码中执行。当进程在执行用户自己的代码时，则称其处于用户运行态，此时处理器在特权级最低的用户代码中运行



> `Binder` 如何进行跨进程通信？

`Binder` 跨进程通信：客户端通过代理对象可以跨进程调用服务端的方法

以应用程序访问系统服务为例，这里主要由四个角色：

* 客户端进程
* 服务端进程
* `Binder` 驱动：负责进程之间 `Binder` 通信的建立、数据在进程之间的传递和交互等一系列底层支持
* `ServiceManager`：维护服务注册表（通常是系统服务），并且可以通过指定的服务名称获取已注册的服务

大致流程如下：

1. 客户端通过 `Binder` 驱动获取服务端代理对象
2. 客户端通过该代理对象向服务端发送请求
3. 代理对象将用户请求通过 `Binder` 驱动发送到服务端
4. 服务端处理用户请求，并通过 `Binder` 驱动返回处理结果



> `Linux` 本来就有多种跨进程机制，为什么还要 `Binder`？

* <u>通信方式</u>：

  基于 `Client-Server` 的通信方式已经在互联网上被广泛应用了，`Android` 系统中，为了向应用开发者提供丰富的功能，这种通信方式更是无处不在，比如：蓝牙、`WIFI`、键盘这些都由不同的 `Server` 负责管理，应用程序只需作为 `Client` 与这些 `Server` 通信即可使用相应的服务。

  在 `Linux` 的跨进程通信方式中，支持 `Client-Server` 的只有 `Socket`，但是 `Socket` 作为一款通用接口，主要用在跨网络的进程间通信和本机上进程间的低速通信，当然也可以在管道、消息队列、共享内存等 `IPC` 的基础之上架设一套协议来实现 `Client-Server`，但这样会增加系统的复杂性。

* <u>性能</u>：

  | IPC                  | 数据拷贝次数 |
  | :------------------- | ------------ |
  | 共享内存             | 0            |
  | Binder               | 1            |
  | Socket/管道/消息队列 | 2            |

* <u>安全</u>：

  传统 `IPC` 机制没有任何安全措施，完全依赖上层协议来保障，`Android` 为每个安装好的应用程序分配独有的 `UID`，故 `UID` 是鉴别进程身份的重要标志，使用传统 `IPC` 只能由用户在数据包里填入`UID/PID`，但这样不可靠，容易被恶意程序利用，可靠的身份标记只有由 `IPC` 机制本身在内核中添加

***

#### AIDL

> 遵循 `AIDL` 规范可以很方便地编写代码来利用 `Binder` 机制进行 `Client` 与 `Server` 的进程间通信

````java
// IChatManager.aidl

/**
 * 遵循规范编写 AIDL 接口
 */
interface IChatManager {
    String getAnswer(String question);
}
````

如上代码编译后会生成相应的 `Java` 接口，整体结构如下：

````java
public interface IChatManager extends IInterface {
    /**
     * 原AIDL接口声明的方法
     */
    public String getAnswer(String question) throws RemoteException;
    
    /**
     * 对于服务端来说，需要继承Stub并重写相应的方法
     * 通过asBinder()向客户端提供该IBinder对象
     */
    public static abstract class Stub extends Binder implements IChatManager {
        private static final String DESCRIPTOR = "IChatManager";
        static final int TRANSACTION_getAnswer = (IBinder.FIRST_CALL_TRANSACTION + 0);

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
        
        /**
         * 对于客户端来说，收到IBinder对象需要调用该方法才能转为IChatManager（强转是不行的）
         */
        public static IChatManager asInterface(IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            // 如果是相同的进程，则返回本地Binder对象
            if (((iin != null) && (iin instanceof IChatManager))) {
                return ((IChatManager) iin);
            }
            // 如果是不同的进程，返回代理对象
            return new IChatManager.Stub.Proxy(obj);
        }

        @Override
        public IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_getAnswer: {
                    data.enforceInterface(descriptor);
                    String _arg0;
                    _arg0 = data.readString();
                    String _result = this.getAnswer(_arg0);
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements IChatManager {
            private IBinder mRemote;

            Proxy(IBinder remote) {
                mRemote = remote;
            }

            @Override
            public IBinder asBinder() {
                return mRemote;
            }

            public String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public String getAnswer(String question) throws RemoteException {
                Parcel _data = Parcel.obtain();
                Parcel _reply = obtain();
                String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(question);
                    boolean _status = mRemote.transact(Stub.TRANSACTION_getAnswer, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }
    }
}
````

`IBinder`：实现了该接口的对象，表示该对象具有跨进程传输的能力

`IInterface`：实现了该接口的对象，表示该对象具有提供跨进程传输对象的能力

```java
public interface IInterface{
    public IBinder asBinder();
}
```

***

#### 参考文章

* [写给 Android 应用工程师的 Binder 原理剖析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/35519585)
* [Binder学习指南 | Weishu's Notes](https://weishu.me/2016/01/12/binder-index-for-newer/)
