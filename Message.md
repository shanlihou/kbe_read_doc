# KBEngine 进程间通信方式

## 废话环节

一款多进程服务端游戏引擎逃不开的一个话题就是他的进程间通信方式.  
KBEngine 不管是引擎层,还是脚本层,都是通过socket作为其进程间通信方式  

## 宏展开

KBEngine 读源码最头疼的应该是就是其宏的使用了,不仅多,并且嵌套层级多.  
进程间通信也是包裹在层层嵌套中,好在这些宏具有一定相似性,大部分是为了解决 c++ 宏的不支持可变参数.  
所以只能定义一堆 args0, args1, args2.  
既然逃不开,索性先挑一个例子,自顶向下解包,抽丝剥茧还原其本来面貌.  
下面是展开的过程,可以跳过,直接看展开后内容

1. 最顶层:

```cpp
#define DBMGR_MESSAGE_DECLARE_STREAM(NAME, MSG_LENGTH)							\
	DBMGR_MESSAGE_HANDLER_STREAM(NAME)											\
	NETWORK_MESSAGE_DECLARE_STREAM(Dbmgr, NAME,									\
				NAME##DbmgrMessagehandler_stream, MSG_LENGTH)					\
																				\
```

2. 现在替换 DBMGR_MESSAGE_HANDLER_STREAM 以及 NETWORK_MESSAGE_DECLARE_STREAM:

DBMGR_MESSAGE_HANDLER_STREAM 原型为:
```cpp

#if defined(NETWORK_INTERFACE_DECLARE_BEGIN)
	#undef DBMGR_MESSAGE_HANDLER_STREAM
#endif

#if defined(DEFINE_IN_INTERFACE)
#if defined(DBMGR)
#define DBMGR_MESSAGE_HANDLER_STREAM(NAME)										\
	void NAME##DbmgrMessagehandler_stream::handle(Network::Channel* pChannel,	\
												KBEngine::MemoryStream& s)		\
	{																			\
			KBEngine::Dbmgr::getSingleton().NAME(pChannel, s);					\
	}																			\

#else
#define DBMGR_MESSAGE_HANDLER_STREAM(NAME)										\
	void NAME##DbmgrMessagehandler_stream::handle(Network::Channel* pChannel,	\
												KBEngine::MemoryStream& s)		\
	{																			\
	}																			\

#endif
#else
#define DBMGR_MESSAGE_HANDLER_STREAM(NAME)										\
	class NAME##DbmgrMessagehandler_stream : public Network::MessageHandler		\
	{																			\
	public:																		\
		virtual void handle(Network::Channel* pChannel,							\
												KBEngine::MemoryStream& s);		\
	};																			\

#endif

```

下面是NETWORK_MESSAGE_DECLARE_STREAM的原型,这个好一点没有那么多条件语句

```cpp
#define NETWORK_MESSAGE_DECLARE_STREAM(DOMAIN, NAME, MSGHANDLER,	\
											MSG_LENGTH)				\
	NETWORK_MESSAGE_HANDLER(DOMAIN, NAME, MSGHANDLER, MSG_LENGTH, 	\
														_stream)	\
	MESSAGE_STREAM(NAME)											\
```

替换后内容如下

```cpp
#define DBMGR_MESSAGE_DECLARE_STREAM(NAME, MSG_LENGTH)							\
	class NAME##DbmgrMessagehandler_stream : public Network::MessageHandler		\
	{																			\
	public:																		\
		virtual void handle(Network::Channel* pChannel,							\
												KBEngine::MemoryStream& s);		\
	};																			\
	NETWORK_MESSAGE_HANDLER(Dbmgr, NAME, NAME##DbmgrMessagehandler_stream, MSG_LENGTH, 	\
														_stream)	\
	MESSAGE_STREAM(NAME)											\
```

3. 继续替换 NETWORK_MESSAGE_HANDLER 以及 MESSAGE_STREAM

NETWORK_MESSAGE_HANDLER 原型如下

```cpp
#ifdef DEFINE_IN_INTERFACE
	//#undef DEFINE_IN_INTERFACE

	#define NETWORK_MESSAGE_HANDLER(DOMAIN, NAME, HANDLER_TYPE, MSG_LENGTH, ARG_N)						\
		HANDLER_TYPE* p##NAME = static_cast<HANDLER_TYPE*>(messageHandlers.add(#DOMAIN"::"#NAME,		\
						 new NAME##Args##ARG_N, MSG_LENGTH, new HANDLER_TYPE));							\
		const HANDLER_TYPE& NAME = *p##NAME;															\

	#define NETWORK_MESSAGE_EXPOSED(DOMAIN, NAME);														\
		bool p##DOMAIN##NAME##_exposed = messageHandlers.pushExposedMessage(#DOMAIN"::"#NAME);			\

#else
	#define NETWORK_MESSAGE_HANDLER(DOMAIN, NAME, HANDLER_TYPE, MSG_LENGTH, ARG_N)						\
		extern const HANDLER_TYPE& NAME;																\

	#define NETWORK_MESSAGE_EXPOSED(DOMAIN, NAME)														\

#endif
```

MESSAGE_STREAM 原型如下

```cpp
#ifdef DEFINE_IN_INTERFACE
#define MESSAGE_STREAM(NAME)
#else
#define MESSAGE_STREAM(NAME)										\
	class NAME##Args_stream : public Network::MessageArgs			\
	{																\
	public:															\
		NAME##Args_stream():Network::MessageArgs(){}				\
		~NAME##Args_stream(){}										\
																	\
		virtual int32 dataSize(void)								\
		{															\
			return NETWORK_VARIABLE_MESSAGE;						\
		}															\
		virtual MessageArgs::MESSAGE_ARGS_TYPE type(void)			\
		{															\
			return MESSAGE_ARGS_TYPE_VARIABLE;						\
		}															\
		virtual void addToStream(MemoryStream& s)					\
		{															\
		}															\
		virtual void createFromStream(MemoryStream& s)				\
		{															\
		}															\
	};																\
				
#endif
```

替换后

```cpp
#define DBMGR_MESSAGE_DECLARE_STREAM(NAME, MSG_LENGTH)							\
	class NAME##DbmgrMessagehandler_stream : public Network::MessageHandler		\
	{																			\
	public:																		\
		virtual void handle(Network::Channel* pChannel,							\
												KBEngine::MemoryStream& s);		\
	};																			\
	extern const NAME##DbmgrMessagehandler_stream& NAME;																\
	class NAME##Args_stream : public Network::MessageArgs			\
	{																\
	public:															\
		NAME##Args_stream():Network::MessageArgs(){}				\
		~NAME##Args_stream(){}										\
																	\
		virtual int32 dataSize(void)								\
		{															\
			return NETWORK_VARIABLE_MESSAGE;						\
		}															\
		virtual MessageArgs::MESSAGE_ARGS_TYPE type(void)			\
		{															\
			return MESSAGE_ARGS_TYPE_VARIABLE;						\
		}															\
		virtual void addToStream(MemoryStream& s)					\
		{															\
		}															\
		virtual void createFromStream(MemoryStream& s)				\
		{															\
		}															\
	};																\
```

## 真实展开后

为了验证我们手动解析的结果是否正确,这里通过生成宏展开后的中间文件 kbe\src\_objs\dbmgr_d\Debug\dbmgr_interface.i 后来对比下.  
打开 dbmgr_interface.i 后发现跟我们手动解析的还是有差异的,原因是我在手动解析的时候默认 DEFINE_IN_INTERFACE 宏是未定义的  
后来我发现在 dbmgr 的 dbmgr_interface.cpp 里面 include 的时候是这么操作的  

```cpp
#include "dbmgr_interface.h"
#define DEFINE_IN_INTERFACE
#define DBMGR
#include "dbmgr_interface.h"
```

所以其实两种情况都实现了,如下是宏展开后的内容

```cpp
//DBMGR_MESSAGE_DECLARE_STREAM
class lookUpEntityByDBIDDbmgrMessagehandler_stream : public Network::MessageHandler
{
public:
	virtual void handle(Network::Channel* pChannel, KBEngine::MemoryStream& s);
};
extern const lookUpEntityByDBIDDbmgrMessagehandler_stream& lookUpEntityByDBID;
class lookUpEntityByDBIDArgs_stream : public Network::MessageArgs
{
public:
	lookUpEntityByDBIDArgs_stream():Network::MessageArgs(){}
	~lookUpEntityByDBIDArgs_stream(){}
	virtual int32 dataSize(void) { return -1; }
	virtual MessageArgs::MESSAGE_ARGS_TYPE type(void) {
			return MESSAGE_ARGS_TYPE_VARIABLE;
	}
	virtual void addToStream(MemoryStream& s) { }
	virtual void createFromStream(MemoryStream& s) { }
	};
//unknown

void lookUpEntityByDBIDDbmgrMessagehandler_stream::handle(Network::Channel* pChannel, KBEngine::MemoryStream& s)
{
	KBEngine::Dbmgr::getSingleton().lookUpEntityByDBID(pChannel, s);
}
lookUpEntityByDBIDDbmgrMessagehandler_stream* plookUpEntityByDBID = static_cast<lookUpEntityByDBIDDbmgrMessagehandler_stream*>(messageHandlers.add("Dbmgr""::""lookUpEntityByDBID", new lookUpEntityByDBIDArgs_stream, -1, new lookUpEntityByDBIDDbmgrMessagehandler_stream));
const lookUpEntityByDBIDDbmgrMessagehandler_stream& lookUpEntityByDBID = *plookUpEntityByDBID;
```

从上面这个可以看出,注册 handle 的地方就是这个 messageHandlers.add 了 这个 messageHandlers 的定义是包裹在宏 NETWORK_INTERFACE_DECLARE_BEGIN 之中

```cpp
#ifndef DEFINE_IN_INTERFACE
#define NETWORK_INTERFACE_DECLARE_BEGIN(INAME) 						\
	namespace INAME													\
{																	\
		extern Network::MessageHandlers messageHandlers;			\

#else
#define NETWORK_INTERFACE_DECLARE_BEGIN(INAME) 						\
	namespace INAME													\
{																	\
		Network::MessageHandlers messageHandlers(#INAME);			\

#endif
```

展开之后发现其类型为 MessageHandler 那么就来看看add函数是怎么实现的.  
看了后发现add函数主要操作msgHandlers_属性,这个map里面保存了msgID 到 msgHandler 的映射  
注册的话,内容大致这么多  

## 消息发送过程  

