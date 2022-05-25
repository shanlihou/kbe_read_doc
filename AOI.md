# AOI

## 0x01: 是什么?

AOI(Area Of Interest) 主要指游戏中可视范围.
对于KBEngine服务端引擎来说,主要用来决定哪些数据对客户端进行同步,并让具有AI的实体只有在范围内存在玩家情况下才活动.  
KBEngine的AOI系统的是依赖一套十字链表实现的.横竖各是一个链表,链表里面的元素按照坐标顺序排列.  
如果游戏不具有高度,两个列表就够了,一个代表坐标轴x,一个代表坐标轴z,如果需要高度,还可以用上坐标轴y

## 0x02: 相关实体

### Space实体

coordinateSystem_(CoordinateSystem): 坐标系统属性  

## 0x03: 坐标系统相关节点类

### CoordinateNode: 坐标系统基本节点

x_: 临时坐标,不代表节点真实坐标,主要用在坐标系统中运算使用
x(): 用来赋值及获取x_
xx(): 真实坐标.

### RangeTriggerNode: 父类为CoordinateNode,通常用于表示AOI的左右或者上下边界,


## 0x04: 实体创建到Space

说到AOI就是离不开Space实体,KBEngine里面有一类特殊的实体,可以理解为场景实体,所有场景相关操作都在Space里面进行(既然是和场景相关,所以Space一定是在Cell进程里创建)  
一个实体的cell部分必须存在于某一个Space实体内部.十字链表也就维护在Space内  
当一个实体初始化之后,会将其插入到这个坐标系统中,代码如下

```cpp
bool CoordinateSystem::insert(CoordinateNode* pNode)
{
    // 前面太长了
	if(isEmpty())
	{
        // ......
        // 坐标系统中每节点,做一些初始化操作
        // 比如初始化x z链表头节点等操作
		return true;
	}

	pNode->old_xx(-FLT_MAX);
	pNode->old_yy(-FLT_MAX);
	pNode->old_zz(-FLT_MAX);


	CoordinateNode* pCurrNode = first_x_coordinateNode_; // 获得x轴链表的头结点
	while (pCurrNode->pNextX() &&
		((pCurrNode->x() < pNode->xx()) ||
		(pCurrNode->x() == pNode->xx() && pNode->hasFlags(COORDINATE_NODE_FLAG_NEGATIVE_BOUNDARY)))) // 这里整个循环的目的是找到第一个pCurrNode 且pNode在pCurrNode的视野范围内
	{
		if (pCurrNode->hasFlags(COORDINATE_NODE_FLAG_NEGATIVE_BOUNDARY) && !pCurrNode->hasFlags(COORDINATE_NODE_FLAG_REMOVED) &&
			static_cast<RangeTriggerNode*>(pCurrNode)->isInXRange(pNode)) // 这里如果当前节点是一个AOI的左边界,并且刚好当前节点在当前整个边界内的话,就直接退出
		{
			break;
		}
		pCurrNode = pCurrNode->pNextX();
	}

	pNode->x(pCurrNode->x()); // 插入开始
	CoordinateNode* pPreNodeX = pCurrNode->pPrevX();
	if (pPreNodeX)
	{
		pPreNodeX->pNextX(pNode);
		pNode->pPrevX(pPreNodeX);
	}
	pCurrNode->pPrevX(pNode);
	pNode->pNextX(pCurrNode);
	if (pCurrNode == first_x_coordinateNode_)
		first_x_coordinateNode_ = pNode; // 插入结束, 这里主要是把pNode插入到pCurrNode前面,因为pCurrNode是第一个能看到pNode的节点

    // ......
    // 这里省略了y坐标轴和z坐标轴的插入相关操作,起始就是x插入相关的复制粘贴
	pNode->pCoordinateSystem(this);
	++size_;

	update(pNode); //下面开始走进入视野相关操作
	return true;
}
```

## 0x04: 更新节点位置及触发进入视野

上面介绍了将节点插入到坐标系统时候发生了什么,可以看到在插入最后调用了update函数  
这个update函数主要作用是当坐标系统中某个节点坐标发生变化时候,更新其在坐标系统中链表的位置,并执行相关进入视野的操作.
由于大部分代码都是重复的,咱们着重于x坐标向着x坐标轴正方向更新时候.

```cpp
void CoordinateSystem::update(CoordinateNode* pNode)
{
	if (pNode->xx() != pNode->old_xx())
	{
		CoordinateNode* pCurrNode = pNode->pPrevX();
		while (pCurrNode && pCurrNode != pNode && // 这个循环主要用来让节点在链表中左移,比如当前节点坐标向x轴负方向更新
			((pCurrNode->x() > pNode->xx()) ||
			(pCurrNode->x() == pNode->xx() && (pNode->hasFlags(COORDINATE_NODE_FLAG_NEGATIVE_BOUNDARY) || pCurrNode->hasFlags(COORDINATE_NODE_FLAG_POSITIVE_BOUNDARY)))\
				))
		{
			moveNodeX(pNode, pNode->xx(), pCurrNode);
			pCurrNode = pNode->pPrevX();
		}

		pCurrNode = pNode->pNextX();
		while (pCurrNode && pCurrNode != pNode && // 这个循环主要用来让节点在链表中右移,比如当前节点坐标向x轴正方向更新
			((pCurrNode->x() < pNode->xx()) || // 这里判断了如果pNode比 pCurrNode大或者 相等情况下 pNode是右边界,或者pCurrNode是左边界就进行更新
			(pCurrNode->x() == pNode->xx() && (pNode->hasFlags(COORDINATE_NODE_FLAG_POSITIVE_BOUNDARY) || pCurrNode->hasFlags(COORDINATE_NODE_FLAG_NEGATIVE_BOUNDARY)))\
				))
		{
			moveNodeX(pNode, pNode->xx(), pCurrNode); // 这里将当前节点向右移动 相当于进行了一次两个节点位置互换 swap(pNode, pCurrNode)
			pCurrNode = pNode->pNextX(); // 因为这里pNode已经移动到前一个pCurrNode右边了,所以这个新的pCurrNode是新的pNode右节点.
		}

		pNode->x(pNode->xx());
	}
    // ......
    // 这里省去了 y和z轴的更新操作
	pNode->resetOld();
}
```

下面是具体的节点在坐标系统中移动相关的代码

```cpp
void CoordinateSystem::moveNodeX(CoordinateNode* pNode, float px, CoordinateNode* pCurrNode)
{// 每次把节点往左或右交换一次swap(2,3) before: 1->2->3->4 end: 1->3->2->4
	if (pCurrNode != NULL)
	{
		pNode->x(pCurrNode->x());

		if (pNode->pPrevX() == pCurrNode)
		{
            // ......
			// 咱们还是以pNode向右为例,所以这里重复代码省去了
		}
		else
		{
			KBE_ASSERT(pCurrNode->x() <= px); // 这里要保证pNode坐标肯定是大约等于pCurrNode,不然交换会出问题
            // 为了方便理解, 1->2->3->4
            // pNode = 2
            // pCurrNode = 3
            // pNextNode = 4
			CoordinateNode* pNextNode = pCurrNode->pNextX();
			if (pNextNode != pNode)
			{
				pCurrNode->pNextX(pNode); // 3->next = 2
				if (pNextNode)
					pNextNode->pPrevX(pNode); // 4->pre = 2

				if (pNode->pPrevX())
					pNode->pPrevX()->pNextX(pNode->pNextX()); // 1->next = 3

				if (pNode->pNextX())
				{
					pNode->pNextX()->pPrevX(pNode->pPrevX()); // 3->pre = 1

					if (pNode == first_x_coordinateNode_)
						first_x_coordinateNode_ = pNode->pNextX(); // 走到这里可能是存在这种情况 1(pNode)->2->3->4(pCurrNode)->5 移动后变成 2->3->4->1->5
				}

				pNode->pPrevX(pCurrNode); // 2->pre = 3
				pNode->pNextX(pNextNode); // 2->next = 4
			}
		}
		// 这里只要是两个位置发生互换都走下互相的onNodePass,再在里面判断是否在范围内或者出范围
		if (!pNode->hasFlags(COORDINATE_NODE_FLAG_HIDE_OR_REMOVED)) // 这里判断如果当前节点属于隐藏节点或者删除节点就不用走onNodePass了 比如边界节点就属于HIDE节点
		{
			pCurrNode->onNodePassX(pNode, true);
		}

		if (!pCurrNode->hasFlags(COORDINATE_NODE_FLAG_HIDE_OR_REMOVED))
		{
			pNode->onNodePassX(pCurrNode, false);
		}
	}
}
```

好了,当一个节点在坐标系统中移动之后,

```cpp
void CoordinateNode::onNodePassX(CoordinateNode* pNode, bool isfront) // 如果是玩家自身节点走到这里是没有逻辑的,只有边界节点触发才会有逻辑
{
}

void RangeTriggerNode::onNodePassX(CoordinateNode* pNode, bool isfront)
{
	if (!hasFlags(COORDINATE_NODE_FLAG_REMOVED) && pRangeTrigger_)
		pRangeTrigger_->onNodePassX(this, pNode, isfront);
}

void RangeTrigger::onNodePassX(RangeTriggerNode* pRangeTriggerNode, CoordinateNode* pNode, bool isfront)
{
	if(pNode == origin())
		return;

	bool wasInZ = pRangeTriggerNode->wasInZRange(pNode);
	bool isInZ = pRangeTriggerNode->isInZRange(pNode);

	// 如果Z轴情况有变化，则Z轴再判断，优先级为zyx，这样才可以保证只有一次enter或者leave
	if(wasInZ != isInZ)
		return;

	bool wasIn = false;
	bool isIn = false;

	if(CoordinateSystem::hasY)
	{
		bool wasInY = pRangeTriggerNode->wasInYRange(pNode);
		bool isInY = pRangeTriggerNode->isInYRange(pNode);

		if(wasInY != isInY)
			return;

		wasIn = pRangeTriggerNode->wasInXRange(pNode) && wasInY && wasInZ;
		isIn = pRangeTriggerNode->isInXRange(pNode) && isInY && isInZ;
	}
	else
	{
		wasIn = pRangeTriggerNode->wasInXRange(pNode) && wasInZ;
		isIn = pRangeTriggerNode->isInXRange(pNode) && isInZ;
	}

	// 如果情况没有发生变化则忽略
	if(wasIn == isIn)
		return;

	if(isIn) //最后根据是进视野还是出视野做出相关操作
	{
		this->onEnter(pNode);
	}
	else
	{
		this->onLeave(pNode);
	}
}
```