# DATABASE


## 创建实体表

KBEngine创建实体表的流程如下
1. 会先创建只含有基本列的表(基本列包含id和sm_autoLoad,如果是子表还会有一个parentID)  
2. 根据def中的内容把所有当前表缺失的列使用ALTER TABLE命令添加进数据库  
3. 把def中不存在但是表中存在的列drop掉  
4. 根据def中同步具有索引的列(INDEX, UNIQUE)  

下面来看下创建实体表相关代码  

```cpp
bool EntityTableMysql::syncToDB(DBInterface* pdbi)
{ // entity_table_mysql.cpp
	if(hasSync())
		return true;

	// DEBUG_MSG(fmt::format("EntityTableMysql::syncToDB(): {}.\n", tableName()));

	char sql_str[SQL_BUF];
	std::string exItems = "";

	if(this->isChild()) // 这里主要判断当前表是否是子表,如果是子表的话要创建父id的列,用来索引对应的父节点
		exItems = ", " TABLE_PARENTID_CONST_STR " bigint(20) unsigned NOT NULL, INDEX(" TABLE_PARENTID_CONST_STR ")";

	kbe_snprintf(sql_str, SQL_BUF, "CREATE TABLE IF NOT EXISTS " ENTITY_TABLE_PERFIX "_%s "
			"(id bigint(20) unsigned AUTO_INCREMENT, PRIMARY KEY idKey (id)%s)" // 创建包含基本列的表
		"ENGINE=" MYSQL_ENGINE_TYPE, 
		tableName(), exItems.c_str());

    try
    {
        bool ret = pdbi->query(sql_str, strlen(sql_str), false);
        if (!ret)
        {
            return false;
        }
    }
    catch (...)
    {
        ERROR_MSG(fmt::format("EntityTableMysql::syncToDB(): error({}: {})\n lastQuery: {}.\n",
            pdbi->getlasterror(), pdbi->getstrerror(), static_cast<DBInterfaceMysql*>(pdbi)->lastquery()));

        return false;
    }

	DBInterfaceMysql::TABLE_FIELDS outs;
    // 这里获取这个表所有域属性,也就是列的相关属性,主要用在后面判断对应的item是否与数据库不统一了,要删除或加新列
    static_cast<DBInterfaceMysql*>(pdbi)->getFields(outs, this->tableName());

	ALL_MYSQL_SET_FLAGS |= NOT_NULL_FLAG;
    // 这里是用来增加上面提到过的sm_autoLoad列
	sync_item_to_db(pdbi, "tinyint not null DEFAULT 0", this->tableName(), TABLE_ITEM_PERFIX"_" TABLE_AUTOLOAD_CONST_STR, 0,
			FIELD_TYPE_TINY, NOT_NULL_FLAG, (void*)&outs, &sync_autoload_item_index);

	EntityTable::TABLEITEM_MAP::iterator iter = tableItems_.begin();
	for(; iter != tableItems_.end(); ++iter)
	{
        // 这里对应给每个item建列
		if(!iter->second->syncToDB(pdbi, (void*)&outs))
			return false;
	}

    std::vector<std::string> dbTableItemNames;

    std::string ttablename = ENTITY_TABLE_PERFIX"_";
    ttablename += tableName();

    pdbi->getTableItemNames(ttablename.c_str(), dbTableItemNames);

    // 检查是否有需要删除的表字段
    std::vector<std::string>::iterator iter0 = dbTableItemNames.begin();
    for (; iter0 != dbTableItemNames.end(); ++iter0)
    {
        std::string tname = (*iter0);

        if (tname == TABLE_ID_CONST_STR ||
            tname == TABLE_ITEM_PERFIX"_" TABLE_AUTOLOAD_CONST_STR ||
            tname == TABLE_PARENTID_CONST_STR)
        {
            continue;
        }

        EntityTable::TABLEITEM_MAP::iterator iter = tableItems_.begin();
        bool found = false;

        for (; iter != tableItems_.end(); ++iter)
        {
            if (iter->second->isSameKey(tname))
            {
                found = true;
                break;
            }
        }

        if (!found)
        {
            if (!pdbi->dropEntityTableItemFromDB(ttablename.c_str(), tname.c_str()))
                return false;
        }
    }

	// 同步表索引
	if (!syncIndexToDB(pdbi))
		return false;

	sync_ = true;
	return true;
}
```

## dbmgr初始化流程

如上,是建立一个实体的数据表流程,具体每个属性的ALTER TABLE是在每个item的syncToDB里面来调用syncItemToDB来实现的.  
如果一个实体的属性是需要建立子表(比如ARRAY这类item)的话,则他的syncToDB里面是空着的,所有逻辑其实是类似父表来走一遍.  
这时候会有疑问:  
这个子表的流程究竟是什么时候走的?  
带着这个疑问来看dbmgr的初始化流程就会发现:  
- dbmgr的初始化时载入所有实体的def信息,并把每个实体初始化为EntityTableMysql类  

```cpp
        bool EntityTables::load(DBInterface* pdbi)
        { // entity_table.cpp
            EntityDef::SCRIPT_MODULES smodules = EntityDef::getScriptModules();
            EntityDef::SCRIPT_MODULES::const_iterator iter = smodules.begin();
            for(; iter != smodules.end(); ++iter)
            {
                ScriptDefModule* pSM = (*iter).get();
                EntityTable* pEtable = pdbi->createEntityTable(this);

                ...

                if (!pEtable->initialize(pSM, pSM->getName()))
                {
                    delete pEtable;
                    return false;
                }

                tables_[pEtable->tableName()].reset(pEtable);
            }

            return true;
        }
```

- 依次对每个 EntityTableMysql 调用初始化接口并将其存入将要用来建表的 map 属性 tables_ 里面
- 在上一步对每个 EntityTableMysql 初始化时对所有 Properties 依次调用初始化,如果当前 property 是 ARRAY 类型,则将其也加入 table_ 里面,并继续递归初始化

```cpp
        bool EntityTableItemMysql_ARRAY::initialize(const PropertyDescription* pPropertyDescription,
                                                    const DataType* pDataType, std::string name)
        {
            ...

            ret = pArrayTableItem->initialize(pPropertyDescription,
                static_cast<FixedArrayType*>(const_cast<DataType*>(pDataType))->getDataType(), itemName.c_str());

            ...

            pTable->pEntityTables()->addTable(pTable);·
            return true;
        }
```

- 根据之前初始化得到的 tables_ 来走上面的 EntityTableMysql::syncToDB 函数

## 实体写DB流程

- 实体在销毁过程开始时候回首先调用引擎的 writeToDB 接口

```cpp
    SCRIPT_METHOD_DECLARE("writeToDB",						pyWriteToDB,						METH_VARARGS,					0)
```

- 之后来到下面这个函数

```cpp
void Entity::writeToDB(void* data, void* extra1, void* extra2)
{
	PyObject* pyCallback = NULL;
	int8 shouldAutoLoad = dbid() <= 0 ? 0 : -1;

	// data 是有可能会NULL的， 比如定时存档是不需要回调函数的
	if(data != NULL)
		pyCallback = static_cast<PyObject*>(data);

	if(extra1 != NULL && (*static_cast<int*>(extra1)) != -1)
		shouldAutoLoad = (*static_cast<int*>(extra1)) > 0 ? 1 : 0;
    
    // ...... 省略一万字
    isArchiveing_ = true; // 归档标记

	CALLBACK_ID callbackID = 0;
	if(pyCallback != NULL)
	{
		callbackID = callbackMgr().save(pyCallback);
	}

	// creatingCell_ 此时可能正在创建cell
	// 不过我们在此假设在cell未创建完成的时候base这个接口被调用
	// 写入数据库的是该entity的初始值， 并不影响
	if(this->cellEntityCall() == NULL) 
	{
		onCellWriteToDBCompleted(callbackID, shouldAutoLoad, -1);
	}
	else
	{
		Network::Bundle* pBundle = Network::Bundle::createPoolObject(OBJECTPOOL_POINT);
		(*pBundle).newMessage(CellappInterface::reqWriteToDBFromBaseapp);
		(*pBundle) << this->id();
		(*pBundle) << callbackID;
		(*pBundle) << shouldAutoLoad;
		sendToCellapp(pBundle);
	}
}
```

- 可以看到当从实体 base 身上调用 writeToDB 时候,会先走到实体的 cell 部分,下面看下 cell 代码

```cpp
void Entity::writeToDB(void* data, void* extra1, void* extra2)
{
	CALLBACK_ID* pCallbackID = static_cast<CALLBACK_ID*>(data);
	CALLBACK_ID callbackID = 0;

	if(pCallbackID)
		callbackID = *pCallbackID;

	int8 shouldAutoLoad = -1;
	if (extra1)
		shouldAutoLoad = *static_cast<int8*>(extra1);

	int dbInterfaceIndex = -1;

    // ...... 省略一万字

	onWriteToDB(); // 这里回调到脚本层
	backupCellData(); // 这里是吧cellData先行发回给base部分,发送协议为Baseapp::onBackupEntityCellData

	Network::Bundle* pBundle = Network::Bundle::createPoolObject(OBJECTPOOL_POINT);
	(*pBundle).newMessage(BaseappInterface::onCellWriteToDBCompleted);
	(*pBundle) << this->id();
	(*pBundle) << callbackID;
	(*pBundle) << shouldAutoLoad;
	(*pBundle) << dbInterfaceIndex;

	if(this->baseEntityCall())
	{
		this->baseEntityCall()->sendCall(pBundle);
	}
}
```

- 看到这里先将 cell 数据打包成 cellData 给 base 先发一份,然后再调用 base 的 onCellWriteToDBCompleted 接口

```cpp
void Entity::onCellWriteToDBCompleted(CALLBACK_ID callbackID, int8 shouldAutoLoad, int dbInterfaceIndex)
{
	SCOPED_PROFILE(SCRIPTCALL_PROFILE);
	CALL_ENTITY_AND_COMPONENTS_METHOD(this, SCRIPT_OBJECT_CALL_ARGS0(pyTempObj, const_cast<char*>("onPreArchive"), GETERR));

	if (dbInterfaceIndex >= 0)
		dbInterfaceIndex_ = dbInterfaceIndex;

	hasDB(true);
	
	onWriteToDB();

	// 如果在数据库中已经存在该entity则允许应用层多次调用写库进行数据及时覆盖需求
	if(this->DBID_ > 0)
		isArchiveing_ = false;

	Components::ComponentInfos* dbmgrinfos = Components::getSingleton().getCachedDbmgr(this->dbid());

	if(dbmgrinfos == NULL || dbmgrinfos->pChannel == NULL || dbmgrinfos->cid == 0)
	{
		ERROR_MSG(fmt::format("{}::onCellWriteToDBCompleted({}): not found dbmgr!\n", 
			this->scriptName(), this->id()));

		return;
	}
	
	MemoryStream* s = MemoryStream::createPoolObject(OBJECTPOOL_POINT);

	try
	{
		addPersistentsDataToStream(ED_FLAG_ALL, s);
	}
	catch (MemoryStreamWriteOverflow & err)
	{
		ERROR_MSG(fmt::format("{}::onCellWriteToDBCompleted({}): {}\n",
			this->scriptName(), this->id(), err.what()));

		MemoryStream::reclaimPoolObject(s);
		return;
	}

	if (s->length() == 0)
	{
		MemoryStream::reclaimPoolObject(s);
		return;
	}

	KBE_SHA1 sha;
	uint32 digest[5];

	sha.Input(s->data(), s->length());
	sha.Result(digest);

	// 检查数据是否有变化，有变化则将数据备份并且记录数据hash，没变化什么也不做
	if (memcmp((void*)&persistentDigest_[0], (void*)&digest[0], sizeof(persistentDigest_)) == 0)
	{
		MemoryStream::reclaimPoolObject(s);

		if (callbackID > 0)
		{
			PyObjectPtr pyCallback = callbackMgr().take(callbackID);
			if (pyCallback != NULL)
			{
				PyObject* pyargs = PyTuple_New(2);

				Py_INCREF(this);
				PyTuple_SET_ITEM(pyargs, 0, PyBool_FromLong(1));
				PyTuple_SET_ITEM(pyargs, 1, this);

				PyObject* pyRet = PyObject_CallObject(pyCallback.get(), pyargs);
				if (pyRet == NULL)
				{
					SCRIPT_ERROR_CHECK();
				}
				else
				{
					Py_DECREF(pyRet);
				}
				Py_DECREF(pyargs);
			}
			else
			{
				ERROR_MSG(fmt::format("{}::onWriteToDBCallback: not found callback:{}.\n",
					this->scriptName(), callbackID));
			}			
		}
		return;
	}
	else
	{
		setDirty((uint32*)&digest[0]);
	}

	Network::Bundle* pBundle = Network::Bundle::createPoolObject(OBJECTPOOL_POINT);
	(*pBundle).newMessage(DbmgrInterface::writeEntity);

	(*pBundle) << g_componentID;
	(*pBundle) << this->id();
	(*pBundle) << this->dbid();
	(*pBundle) << this->dbInterfaceIndex();
	(*pBundle) << this->lastDbmgrCid();
	(*pBundle) << this->pScriptModule()->getUType();
	(*pBundle) << callbackID;
	(*pBundle) << shouldAutoLoad;

	// 记录登录地址
	if(this->dbid() == 0)
	{
		uint32 ip = 0;
		uint16 port = 0;
		
		if(this->clientEntityCall())
		{
			ip = this->clientEntityCall()->addr().ip;
			port = this->clientEntityCall()->addr().port;
		}

		(*pBundle) << ip;
		(*pBundle) << port;
	}

	(*pBundle).append(*s);

	dbmgrinfos->pChannel->send(pBundle);
	MemoryStream::reclaimPoolObject(s);
	lastDbmgrCid(dbmgrinfos->cid);
}
```