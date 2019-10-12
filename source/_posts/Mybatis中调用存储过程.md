---
title: Mybatis中调用存储过程
date: 2019-10-12 14:57:54
tags:
- Mybatis
- 存储过程
- Oracle
---

很多人对Mybatis调用存储过程模棱两可，不知道该怎么配置，本篇将接受Mybatis如何调用Oracle存储过程（带有入参和出参）

## 语法
需要注意的是调用存储过程的方法应当是select标签，statementType=CALLABLE，这里的parameterType建议必须填写，以免出现问题。
```xml
<select id="test" statementType="CALLABLE" parameterType="com.juncheng.entity.TestEntity">
    {CALL TEST(#{aa,mode=IN},#{bb,mode=OUT,jdbcType=CURSOR,resultMap=emp},#{cc})}
</select>
```
参数的mode标识当前参数是入参还是出参，入参使用IN，出参使用OUT，如果即是入参也是出参，可以不用填写

## 返回值
大多数情况下，调用一个存储过程过后，会有返回值，并且这个返回值可能是查询列表，通过OUT参数输出，这个时候存储过程的参数类型应该是游标（SYS_REFCURSOR）类型，在Mybatis中
参数的jdbcType=CURSOR，并且需要指定该列表的映射实体resultMap=emp，resultMap代码如下：
```xml
<resultMap id="emp" type="com.juncheng.entity.EmpEntity">
    <id column="EMPNO" property="empno" jdbcType="INTEGER"/>
    <result column="ENAME" property="ename" jdbcType="VARCHAR"/>
    <result column="JOB" property="job" jdbcType="VARCHAR"/>
    <result column="MGR" property="mgr" jdbcType="INTEGER"/>
    <result column="HIREDATE" property="hiredate" jdbcType="TIMESTAMP"/>
    <result column="SAL" property="sal" jdbcType="INTEGER"/>
    <result column="COMM" property="comm" jdbcType="INTEGER"/>
    <result column="DEPTNO" property="deptno" jdbcType="INTEGER"/>
</resultMap>
```
OUT参数并不是直接通过Mapper接口返回值返回，而是set到参数对象对应的属性中，一般调用存储过程是没有返回值的，所有的OUT参数都返回到Mapper接口的参数中。代码如下：
```java
public interface TestMapper {
    void test(TestEntity entity);
}


public class TestEntity {
    private Integer aa;
    private List<EmpEntity> bb;
    private Integer cc;
    
    //setter getter...
}

public class EmpEntity {

    private Integer empno;
    private String ename;
    private String job;
    private Integer mgr;
    private Date hiredate;
    private Integer sal;
    private Integer comm;
    private Integer deptno;
    
    //setter getter...
}
```
mybatis-conf.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
<settings>
    <setting name="mapUnderscoreToCamelCase" value="true" />
</settings>
<environments default="development">
    <environment id="development">
        <transactionManager type="JDBC" />
        <!-- 配置数据库连接信息 -->
        <dataSource type="POOLED">
            <property name="driver" value="oracle.jdbc.driver.OracleDriver" />
            <property name="url" value="jdbc:oracle:thin:@127.0.0.1:1521:ORCL" />
            <property name="username" value="scott" />
            <property name="password" value="scott" />
        </dataSource>
    </environment>
</environments>
<mappers>
    <mapper resource="com/juncheng/mapper/TestMapper.xml"></mapper>
</mappers>
</configuration>
```
测试用例，主入口
```java
public class MybatisOracleTest {
    public static void main(String[] args) throws IOException {
        String resource = "mybatis-conf.xml";
        //1.流形式读取mybatis配置文件
        InputStream stream = Resources.getResourceAsStream(resource);
        //2.通过配置文件创建SqlSessionFactory
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(stream);
        SqlSession sqlSession = factory.openSession();
        //以下是测试
        TestMapper testMapper = sqlSession.getMapper(TestMapper.class);
        TestEntity entity = new TestEntity();
        entity.setAa(1);
        entity.setCc(2);
        testMapper.test(entity);
        System.out.println(entity);

    }
}
```