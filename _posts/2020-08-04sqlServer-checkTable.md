判断表是否存在，不存在则新建表

    @Autowired
    EntityManager em;
    String tableInfoSql = "SELECT  * FROM dbo.SysObjects WHERE ID = object_id(N'[event_"+ yearMonth + "]') AND OBJECTPROPERTY(ID, 'IsTable') = 1";
        List tableInfo = em.createNativeQuery(tableInfoSql).getResultList();
        if(CollectionUtils.isEmpty(tableInfo)){
            String createTableSql = "***";
            em.createNativeQuery(createTableSql).executeUpdate();
        }