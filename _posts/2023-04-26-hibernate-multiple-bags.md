---
layout: post
title: multiple bags 问题解决
categories: [Java]
---

在修改OneToMany联表查询的时候，声明字段为List后面在项目启动的时候报错：hibernate: cannot simultaneously fetch multiple bags

详细问题信息

```
org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags at org.hibernate.loader.BasicLoader.postInstantiate(BasicLoader.java:94) at org.hibernate.loader.entity.EntityLoader.<init>(EntityLoader.java:119) at 
org.hibernate.loader.entity.EntityLoader.<init>(EntityLoader.java:71) at org.hibernate.loader.entity.EntityLoader.<init>(EntityLoader.java:54) at org.hibernate.loader.entity.BatchingEntityLoader.createBatchingEntityLoader(BatchingEntityLoader.java:133) 
at org.hibernate.persister.entity.AbstractEntityPersister.createEntityLoader(AbstractEntityPersister.java:1914) at org.hibernate.persister.entity.AbstractEntityPersister.createEntityLoader(AbstractEntityPersister.java:1937) at 
org.hibernate.persister.entity.AbstractEntityPersister.createLoaders(AbstractEntityPersister.java:3205) at org.hibernate.persister.entity.AbstractEntityPersister.postInstantiate(AbstractEntityPersister.java:3191) at org.hibernate.impl.SessionFactoryImpl.<init>(SessionFactoryImpl.java:348) at
 org.hibernate.cfg.Configuration.buildSessionFactory(Configuration.java:1872) at org.hibernate.ejb.Ejb3Configuration.buildEntityManagerFactory(Ejb3Configuration.java:906) at org.hibernate.ejb.HibernatePersistence.createEntityManagerFactory(HibernatePersistence.java:57) at javax.persistence.
 Persistence.createEntityManagerFactory(Persistence.java:63)
```

问题重现步骤：

```
当一个实体对象中包含多于一个non-lazy获取策略时，比如@OneToMany，@ManyToMany或者@ElementCollection时，获取策略为(fetch = FetchType.EAGER)
```

问题分析：

当（fetch=FetchType.EAGER）时，持久框架抓取一方的对象，会将关联的多方的对象加载到容器中，关联的对象又有可能关联其他多方对象，Hibernate实现的JPA，默认最高抓取深度含本身为四级（又有个属性配置是0-3），若多方（第二级）存在重复的值，则在第三级中抓取的值就无法映射（获取的值太多？）就会出现multiple bags

解决方案：

1. **将(fetch = FetchType.EAGER)改为(fetch = FetchType.LAZY)**
2. **将List修改成Set集合，即推荐@ManyToMany或@OneToMany的Many方此时用Set容器来存放，而不用List集合。**
3. **改变FetchMode为@Fetch(FetchMode.SUBSELECT)，即发送另外一条select语句抓取前面查询到的所有实体对象的关联实体。**（Hibernate特有的，非JPA标准）
4. **在对应的属性上添加@IndexColumn，该注解允许你指明存放索引值的字段，目的跟Set容器不允许重复元素的道理一样。**（Hibernate特有的，非JPA标准）