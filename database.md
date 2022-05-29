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

```cpp
	static bool writeDB(DB_TABLE_OP optype, DBInterface* pdbi, mysql::DBContext& context, DBidCache* pDbidCache, bool hasDbidCache, bool writeCurContext = true)
	{
		bool ret = true;

		if (!context.isEmpty && writeCurContext)
		{
			SqlStatement* pSqlcmd = createSql(pdbi, optype, context.tableName,
				context.parentTableDBID,
				context.dbid, context.items);

			ret = pSqlcmd->query();  // 这边执行一次写入操作
			context.dbid = pSqlcmd->dbid();
			if (pDbidCache)
			{
				pDbidCache->setDbid(pSqlcmd->dbid());
			}
			delete pSqlcmd;
		}

		if (optype == TABLE_OP_INSERT)
		{
			// 开始更新所有的子表
			mysql::DBContext::DB_RW_CONTEXTS::iterator iter1 = context.optable.begin();
			for (; iter1 != context.optable.end(); ++iter1)
			{
				mysql::DBContext& wbox = *iter1->second.get();

				// 绑定表关系
				wbox.parentTableDBID = context.dbid;

				DBidCache* pChildDbidCache = NULL;

				if (pDbidCache && !wbox.isEmpty)
				{
					pChildDbidCache = new DBidCache(wbox.dbid, iter1->first, pDbidCache);
					pDbidCache->addChildDbidCache(pChildDbidCache);
				}

				// 更新子表
				writeDB(optype, pdbi, wbox, pChildDbidCache, hasDbidCache);
			}
		}
		else
		{
			// 如果有父ID首先得到该属性数据库中同父id的数据有多少条目， 并取出每条数据的id
			// 然后将内存中的数据顺序更新至数据库， 如果数据库中有存在的条目则顺序覆盖更新已有的条目， 如果数据数量
			// 大于数据库中已有的条目则插入剩余的数据， 如果数据少于数据库中的条目则删除数据库中的条目
			// select id from tbl_SpawnPoint_xxx_values where parentID = 7;
			KBEUnordered_map< std::string, std::vector<DBID> > childTableDBIDs;
			TABLE_DATAS_MAP  childTableDatas;

			if (hasDbidCache && pDbidCache)
				childTableDBIDs = pDbidCache->childTableDBIDs_;

			if (context.dbid > 0)
			{
				mysql::DBContext::DB_RW_CONTEXTS::iterator iter1 = context.optable.begin();
				for (; iter1 != context.optable.end(); ++iter1) // 遍历子表
				{
					mysql::DBContext& wbox = *iter1->second.get(); // 取出子表context

					KBEUnordered_map<std::string, std::vector<DBID> >::iterator iter =
						childTableDBIDs.find(context.tableName); // 查找 父表 是否有cache

					if (iter == childTableDBIDs.end())
					{
						std::vector<DBID> v; 
						childTableDBIDs.insert(std::pair< std::string, std::vector<DBID> >(wbox.tableName, v)); // 插入子表空 vec

						TABLE_DATAS dataVec;
						childTableDatas.insert(std::pair < std::string, TABLE_DATAS>(wbox.tableName, dataVec)); // 插入子表空 vec
					}

					childTableDatas[wbox.tableName].push_back(iter1->second);
				}

				if (hasDbidCache && pDbidCache)
				{
					if (g_kbeSrvConfig.isCheckDbidCache())
					{
						KBEUnordered_map< std::string, std::vector<DBID> >::iterator tabiter = childTableDBIDs.begin();
						for (; tabiter != childTableDBIDs.end(); tabiter++)
						{
							char sqlstr[MAX_BUF * 10];
							kbe_snprintf(sqlstr, MAX_BUF * 10, "select id from " ENTITY_TABLE_PERFIX "_%s where " TABLE_PARENTID_CONST_STR "=%" PRDBID,
								tabiter->first.c_str(),
								context.dbid);

							if (pdbi->query(sqlstr, strlen(sqlstr), false))
							{
								std::vector<DBID> childDbids;
								MYSQL_RES * pResult = mysql_store_result(static_cast<DBInterfaceMysql*>(pdbi)->mysql());
								if (pResult)
								{
									MYSQL_ROW arow;
									while ((arow = mysql_fetch_row(pResult)) != NULL)
									{
										DBID old_dbid;
										StringConv::str2value(old_dbid, arow[0]);
										childDbids.push_back(old_dbid);
									}

									mysql_free_result(pResult);

									std::sort(childDbids.begin(), childDbids.end());
									std::sort(tabiter->second.begin(), tabiter->second.end());
									if (childDbids != tabiter->second)
									{
										ERROR_MSG(fmt::format("dbidCache is error {}, {}\n", tabiter->first.c_str(), context.dbid));
										for (size_t i = 0; i < childDbids.size(); i++) {
											int d = childDbids[i];
											ERROR_MSG(fmt::format("childDbids {}, {}\n", tabiter->first.c_str(), d));
										}
										for (size_t i = 0; i < tabiter->second.size(); i++) {
											int d = tabiter->second[i];
											ERROR_MSG(fmt::format("tabiter->second {}, {}\n", tabiter->first.c_str(), d));
										}// 这里纯粹检查报个错,然后把因为错了的重新赋值一下
										tabiter->second = childDbids;
										hasDbidCache = false;
										DBidCache* pRootDbidCache = pDbidCache->getRootDbidCache();
										pRootDbidCache->setIsValid(false);
										pDbidCache = NULL;
									}
								}								
							}
						}
					}
				}
				else
				{
					KBEUnordered_map< std::string, std::vector<DBID> >::iterator tabiter = childTableDBIDs.begin();
					for (; tabiter != childTableDBIDs.end(); tabiter++)
					{
						char sqlstr[MAX_BUF * 10];
						kbe_snprintf(sqlstr, MAX_BUF * 10, "select id from " ENTITY_TABLE_PERFIX "_%s where " TABLE_PARENTID_CONST_STR "=%" PRDBID,
							tabiter->first.c_str(),
							context.dbid);

						if (pdbi->query(sqlstr, strlen(sqlstr), false))
						{
							MYSQL_RES * pResult = mysql_store_result(static_cast<DBInterfaceMysql*>(pdbi)->mysql());
							if (pResult)
							{
								MYSQL_ROW arow;
								while ((arow = mysql_fetch_row(pResult)) != NULL)
								{
									DBID old_dbid;
									StringConv::str2value(old_dbid, arow[0]); 
									if (!hasDbidCache)
										tabiter->second.push_back(old_dbid);

									if (!hasDbidCache && pDbidCache)
									{
										DBidCache* pChildDbidCache = new DBidCache(old_dbid, tabiter->first, pDbidCache);
										pDbidCache->addChildDbidCache(pChildDbidCache);
									}
								}

								mysql_free_result(pResult);
							}
						}
					}
				}
			}

			// 如果是要清空此表， 则循环N次已经找到的dbid， 使其子表中的子表也能有效删除
			if (!context.isEmpty)
			{
				// 开始更新所有的子表
				TABLE_DATAS_MAP::iterator tableIter = childTableDatas.begin();
				for (; tableIter != childTableDatas.end(); ++tableIter)
				{
					TABLE_DATAS& datas = tableIter->second;
					KBEUnordered_map<std::string, std::vector<DBID> >::iterator iter =
						childTableDBIDs.find(tableIter->first);

					if (iter != childTableDBIDs.end())
					{
						int nBatchlyWrite = writeDBBatchly(optype, pdbi, context, tableIter->first, datas, iter->second, pDbidCache, hasDbidCache);
						if (nBatchlyWrite >= 0)
						{
							iter->second.erase(iter->second.begin(), iter->second.begin() + nBatchlyWrite);
							for (size_t i = nBatchlyWrite;i < datas.size(); ++i)
							{
								mysql::DBContext& childContext = *(datas[i].get());
								childContext.parentTableDBID = context.dbid;
								childContext.dbid = 0;
								DBidCache* pChildDbidCache = NULL;
								if (pDbidCache && !childContext.isEmpty)
								{
									pChildDbidCache = new DBidCache(childContext.dbid, childContext.tableName, pDbidCache);
									pDbidCache->addChildDbidCache(pChildDbidCache);
								}
								writeDB(optype, pdbi, childContext, pChildDbidCache, hasDbidCache);
							}
						}
					}
				}

				/*mysql::DBContext::DB_RW_CONTEXTS::iterator iter1 = context.optable.begin();
				for(; iter1 != context.optable.end(); ++iter1)
				{
					mysql::DBContext& wbox = *iter1->second.get();

					if(wbox.isEmpty)
						continue;

					// 绑定表关系
					wbox.parentTableDBID = context.dbid;

					KBEUnordered_map<std::string, std::vector<DBID> >::iterator iter =
						childTableDBIDs.find(wbox.tableName);

					if(iter != childTableDBIDs.end())
					{
						if(iter->second.size() > 0)
						{
							wbox.dbid = iter->second.front();
							iter->second.erase(iter->second.begin());
						}

						if(iter->second.size() <= 0)
						{
							childTableDBIDs.erase(wbox.tableName);
						}
					}

					// 更新子表
					writeDB(optype, pdbi, wbox);
				}*/
			}

			// 删除废弃的数据项
			KBEUnordered_map< std::string, std::vector<DBID> >::iterator tabiter = childTableDBIDs.begin();
			for (; tabiter != childTableDBIDs.end(); ++tabiter)
			{
				if (tabiter->second.size() == 0)
					continue;

				// 先删除数据库中的记录
				std::string sqlstr = "delete from " ENTITY_TABLE_PERFIX "_";
				sqlstr += tabiter->first;
				sqlstr += " where " TABLE_ID_CONST_STR " in (";

				std::vector<DBID>::iterator iter = tabiter->second.begin();
				for (; iter != tabiter->second.end(); ++iter)
				{
					DBID dbid = (*iter);

					char sqlstr1[MAX_BUF];
					kbe_snprintf(sqlstr1, MAX_BUF, "%" PRDBID, dbid);
					sqlstr += sqlstr1;
					sqlstr += ",";
				}

				sqlstr.erase(sqlstr.size() - 1);
				sqlstr += ")";
				bool ret = pdbi->query(sqlstr.c_str(), sqlstr.size(), false);
				KBE_ASSERT(ret);

				TABLE_DATAS_MAP::iterator it = childTableDatas.find(tabiter->first);
				if (it != childTableDatas.end() && !it->second.empty())
				{
					mysql::DBContext& wbox = *it->second[0].get();
					auto iter = tabiter->second.begin();
					for (; iter != tabiter->second.end(); ++iter)
					{
						DBID dbid = (*iter);

						wbox.parentTableDBID = context.dbid;
						wbox.dbid = dbid;
						wbox.isEmpty = true;

						DBidCache* pChildDbidCache = NULL;
						if (pDbidCache)
						{
							pChildDbidCache = pDbidCache->getChildDbidCache(tabiter->first, dbid);
						}
						// 删除子表
						writeDB(optype, pdbi, wbox, pChildDbidCache, hasDbidCache);
						if (pChildDbidCache && pDbidCache)
						{
							pDbidCache->delChildDbidCache(pChildDbidCache);
						}
					}
				}
			}
		}
		return ret;
	}
```
