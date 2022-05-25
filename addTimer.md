# addTimer

## 0x01: 添加timer

首先是base 添加timer的流程
```cpp
PyObject* Baseapp::__py_addTimer(PyObject* self, PyObject* args)
{
	float interval, repeat;
	PyObject *pyCallback;

	if (!PyArg_ParseTuple(args, "ffO", &interval, &repeat, &pyCallback))
		S_Return;

	if (!PyCallable_Check(pyCallback))
	{
		PyErr_Format(PyExc_TypeError, "KBEngine::addTimer: '%.200s' object is not callable",
			(pyCallback ? pyCallback->ob_type->tp_name : "NULL"));

		PyErr_PrintEx(0);
		S_Return;
	}
    //上面是传参的转化,因为是从python调用过来得
	ScriptTimers * pTimers = &scriptTimers(); //这里是获得自己引用的kbeScriptTimers_属性的指针 $(9)
	ScriptTimerHandler *handler = new ScriptTimerHandler(pTimers, pyCallback); //这里构造一个ScriptTimerHandler对象 $(8)

	ScriptID id = ScriptTimersUtil::addTimer(&pTimers, interval, repeat, 0, handler);

	if (id == 0)
	{
		delete handler;
		PyErr_SetString(PyExc_ValueError, "Unable to add timer");
		PyErr_PrintEx(0);
		S_Return;
	}

	return PyLong_FromLong(id);
}
```

然后ScriptTimersUtil::addTimer其实就是如下函数包装了一层,就不展示了

```cpp
ScriptID ScriptTimers::addTimer( float initialOffset,
		float repeatOffset, int userArg, TimerHandler * pHandler )
{
	if (initialOffset < 0.f)
	{
		WARNING_MSG(fmt::format("ScriptTimers::addTimer: Negative timer offset ({})\n",
				initialOffset));

		initialOffset = 0.f;
	}

	KBE_ASSERT( g_pApp );

	int hertz = g_kbeSrvConfig.gameUpdateHertz();
	int initialTicks = GameTime( g_pApp->time() + initialOffset * hertz ); //计算第一次触发式后的tick值
	int repeatTicks = 0;

	if (repeatOffset > 0.f)
	{
		repeatTicks = GameTime( repeatOffset * hertz );
		if (repeatTicks < 1)
		{
			repeatTicks = 1;
		}
	}

	uint64 createTime = timestamp();

	TimerHandle timerHandle = g_pApp->timers().add(//调用app的add函数, 下面的代码块中将包含add具体实现 $(1)
			initialTicks, repeatTicks, createTime,
			pHandler, (void *)(intptr_t)userArg );

	if (timerHandle.isSet())//这边应该是一定能走到, $(2)
	{
		int id = this->getNewID(); // $(3)

		map_[ id ] = timerHandle;

		return id;
	}

	return 0;
}
```

如上是ScriptTimers的addTimer实现,然后看下这个

```cpp
template <class TIME_STAMP>
TimerHandle TimersT< TIME_STAMP >::add( TimeStamp startTime, // $(1)
		TimeStamp interval, uint64 createTime, TimerHandler * pHandler, void * pUser )
{
	Time * pTime = new Time( *this, startTime, interval, createTime, pHandler, pUser );// $(4)
	timeQueue_.push( pTime );// 这个timeQueue是个封装的堆,里面是 std::push_heap相关操作
	return TimerHandle( pTime );//这里把pTime包一层,传出去,主要用来cancel相关操作,在调用层加上timerId $(2)
}
```

下面的是生成新的timerId接口,就是找到第一个可用ID

```cpp
ScriptID ScriptTimers::getNewID() // $(3)
{
	ScriptID id = 1;

	while (map_.find( id ) != map_.end())
	{
		++id;
	}

	return id;
}
```

如下是Time的构造函数

```cpp
template <class TIME_STAMP>
TimersT< TIME_STAMP >::Time::Time( TimersBase & owner,// $(4)
		TimeStamp startTime, TimeStamp interval, uint64 createTime,
		TimerHandler * _pHandler, void * _pUser ) :
	TimeBase(owner, _pHandler, _pUser), //这里的_pUser就是从上面传下来的userData,最终存在TimeBase类的属性pUserData_里面
	time_(startTime),
	interval_(interval),
	createTime_(createTime)
{
}
```

下面看下TimersT类的process函数

```cpp
template <class TIME_STAMP>
int TimersT< TIME_STAMP >::process(TimeStamp now)
{
	int numFired = 0;

	while ((!timeQueue_.empty()) && (// 优先级队列非空
		timeQueue_.top()->time() <= now || //且目标时间已经超过头部时间
		timeQueue_.top()->isCancelled())) // 或者头部已经被cancel了
	{
		Time * pTime = pProcessingNode_ = timeQueue_.top();
		timeQueue_.pop();
        //把头部pop出来,然后如果没有cancel,就处理了
		if (!pTime->isCancelled())
		{
			++numFired;
			pTime->triggerTimer();// $(5)
		}

		if (!pTime->isCancelled())
		{
			timeQueue_.push( pTime );
		}
		else
		{
			delete pTime;

			KBE_ASSERT( numCancelled_ > 0 );
			--numCancelled_;
		}
	}

	pProcessingNode_ = NULL;
	lastProcessTime_ = now;
	return numFired;
}
```

如下是trigger函数具体实现

```cpp
template <class TIME_STAMP>
void TimersT< TIME_STAMP >::Time::triggerTimer()// $(5)
{
	if (!this->isCancelled())
	{
		state_ = TIME_EXECUTING;

		pHandler_->handleTimeout( TimerHandle( this ), pUserData_ );

		if ((interval_ == 0) && !this->isCancelled())
		{
			this->cancel();//cancel流程在下面 $(6)
		}
	}

	if (!this->isCancelled())
	{
		time_ += interval_;
		state_ = TIME_PENDING;
	}
}
```

## 0x02: cancel流程
如上,一个timer就加好了,下面来看下cancel的流程

```cpp
inline void TimeBase::cancel()
{
	if (this->isCancelled()){
		return;
	}

	KBE_ASSERT((state_ == TIME_PENDING) || (state_ == TIME_EXECUTING));
	state_ = TIME_CANCELLED;

	if (pHandler_){
		pHandler_->release(TimerHandle(this), pUserData_);//这里调用到release $(7), 这个pHandler就是 $(8) 一路传下来的
		pHandler_ = NULL;
	}

	owner_.onCancel();
}
```

然后来看看release是怎么走的,因为pHandler就是ScriptTimerHandler,而ScriptTimerHandler继承自TimerHandler

```cpp
class TimerHandler
{
	virtual void onRelease( TimerHandle handle, void * pUser ) {
	}
	void release( TimerHandle handle, void * pUser )
	{
		this->decTimerRegisterCount();
		this->onRelease( handle, pUser );
	}
}
class ScriptTimerHandler : public TimerHandler
	virtual void onRelease(TimerHandle handle, void * /*pUser*/)
	{
		ascriptTimers_->releaseTimer(handle); //这里看到他会调到ascriptTimers_的releaseTimer,那么这个ascriptTimers_就是$(9)
		delete this;
	}
}
```

那么现在需要看ScriptTimers的releaseTimer怎么实现的,这个对应咱们刚才调用过的被包装的addTimer

```cpp
void ScriptTimers::releaseTimer( TimerHandle handle )
{
	int numErased = 0;

	Map::iterator iter = map_.begin();

	while (iter != map_.end()) //可以看出这里release的时候效率还是有点低的,会把整个timer都遍历一遍,完全没利用到timerId这个东东
	{
		KBE_ASSERT( iter->second.isSet() );

		if (handle == iter->second)
		{
			map_.erase( iter++ );
			++numErased;
		}
		else
		{
			iter++;
		}
	}

	KBE_ASSERT( numErased == 1 );
}
```

## 0x03: 怎么保证服务器tick频率

类似baseapp和cellapp都是通过如下方式增加每个tick周期的定时器.

```cpp
bool PythonApp::initializeEnd()
{
	gameTickTimerHandle_ = this->dispatcher().addTimer(1000000 / g_kbeSrvConfig.gameUpdateHertz(), this,//这里应该是通过1000000 / g_kbeSrvConfig.gameUpdateHertz() 得到每个tick所需要的周期
		reinterpret_cast<void *>(TIMEOUT_GAME_TICK));

	return true;
}
```