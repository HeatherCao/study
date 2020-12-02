判断表是否存在，不存在则新建表

    @Autowired
    EntityManager em;
    String tableInfoSql = "SELECT  * FROM dbo.SysObjects WHERE ID = object_id(N'[event_"+ yearMonth + "]') AND OBJECTPROPERTY(ID, 'IsTable') = 1";
        List tableInfo = em.createNativeQuery(tableInfoSql).getResultList();
        if (CollectionUtils.isEmpty(tableInfo)) {
            String createTableSql = "***";
            em.createNativeQuery(createTableSql).executeUpdate();
        }
        
    
    // 参数替代化
    // sqlserver中U = 用户表，还有其他的，例如：V = 视图，TF = 表函数，duP = 存储过程，L = 日志等

    String tableInfoSql = "SELECT * FROM dbo.SysObjects WHERE name = ? and type='U'";
        Query nativeQuery = em.createNativeQuery(tableInfoSql);
        nativeQuery.setParameter(1, tableName);
        List tableInfo = nativeQuery.getResultList();
        if (CollectionUtils.isEmpty(tableInfo)) {
            String createTableSql = "***";
            em.createNativeQuery(createTableSql).executeUpdate();
        }
