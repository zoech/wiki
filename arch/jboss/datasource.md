jboss中添加数据源，在web项目中使用EJB的话可以通过persistence.xml配置web项目使用的数据源

1. jboss中添加数据源之后，可以在EJB项目中通过自己代码获取数据源的连接

``` xml
     <local-tx-datasource>
        <jndi-name>platform</jndi-name>
        <connection-url>jdbc:oracle:thin:@59.65.233.199:1521:GIS</connection-url>
        <driver-class>oracle.jdbc.driver.OracleDriver</driver-class>
        <user-name>PLATFORM</user-name>
        <password>PLATFORM</password>
        <exception-sorter-class-name>org.jboss.resource.adapter.jdbc.vendor.OracleExceptionSorter</exception-sorter-class-name>        
          <metadata>
             <type-mapping>Oracle9i</type-mapping>
          </metadata>
      </local-tx-datasource>
```

下面代码可以正确获取改数据源
``` java
	public Connection getConnection() {
 
		Connection connection = null;
		try {
			Context ctx = new InitialContext();// 得到初始化上下文
			DataSource ds = (DataSource) ctx.lookup("java:platform");// 这样查找数据源，不要lookup("platform");
			connection = ds.getConnection();
		} catch (SQLException e) {
			e.printStackTrace();
		} catch (NamingException e) {
			e.printStackTrace();
		}
 
		return connection;
	}
```

1. 要注意在jboss数据源配置的时候，有一个属性“use-java-context”
数据源定义文件**-ds.xml中，有个use-java-context属性，把它设为false的话，JNDIView里生成的数据源就会位于全局命名空间，没有java前缀了。

比如下面这个配置的数据源，和上面的有不同
``` xml
  <local-tx-datasource>
    <jndi-name>flow</jndi-name>
    <!-- 注意下面这个属性配置改为false了 -->
    <use-java-context>false</use-java-context>
    <connection-url>jdbc:oracle:thin:@59.65.233.199:1521:GIS</connection-url>
    <driver-class>oracle.jdbc.driver.OracleDriver</driver-class>
    <user-name>FLOW</user-name>
    <password>FLOW</password>
    <exception-sorter-class-name>org.jboss.resource.adapter.jdbc.vendor.OracleExceptionSorter</exception-sorter-class-name>
    <metadata>
        <type-mapping>Oracle9i</type-mapping>
    </metadata>
  </local-tx-datasource>
```

下面代码是对应的datasource获取，注意ctx.lookup()方法里的数据源名称没有了"java"前缀：
``` java
	public Connection getConnection1() {
		Connection connection = null;
		try {
			Context ctx = new InitialContext();
			DataSource ds = (DataSource) ctx.lookup("flow");
			connection = ds.getConnection();
		} catch (SQLException e) {
			e.printStackTrace();
		} catch (NamingException e) {
			e.printStackTrace();
		}
 
		return connection;
	}
```

如果是在persistence.xml中配置，则可以配置为下面这样：
``` xml
<persistence>
	<persistence-unit name="test">
		<jta-data-source>java:/platform</jta-data-source>
		<properties>
			<property name="hibernate.dialect" value="org.hibernate.dialect.Oracle9Dialect" />
			<property name="com.arjuna.ats.jta.allowMultipleLastResources"
				value="true" />
			<property name="hibernate.hbm2ddl.auto" value="update" />
		</properties>
	</persistence-unit>
 
	<persistence-unit name="test1">
		<jta-data-source>flow</jta-data-source>
		<properties>
			<property name="hibernate.dialect" value="org.hibernate.dialect.Oracle9Dialect" />
			<property name="com.arjuna.ats.jta.allowMultipleLastResources"
				value="true" />
			<property name="hibernate.hbm2ddl.auto" value="update" />
		</properties>
	</persistence-unit>
</persistence>
```
