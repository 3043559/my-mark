描述
用于数据持久化的框架
配置
框架配置
1.读取属性配置文件xxx.properties
\<properties resource="jdbc.properties"/\>
2.设置日志在控制台输出 (数据源+事务提交状态+SQL+入参+出参+连接池等信息)
\<settings\>
\<setting name="logImpl" value="STDOUT_LOGGING"/\>
\</settings\>
3.注册实体类别名
\<typeAliases\>
\<!-- 单个注册(实体类多时，都需依次注册)--\>

\<!-- \<typeAlias type="org.dhrj.zs.entity.Student" alias="student"/\>--\>

\<!-- 批量注册，别名规范 = 实体类名驼峰格式--\>

\<package name="com......entity"/\>
\</typeAliases\>
4.配置数据库连接 default：通过id值去指定数据库配置；然后新建环境配置，在配置中添加事务管理器和数据源
\<environments default="test"\>

\<environment id="company"\>

\<!--

配置事务管理器

type：指定事务管理的方式

"JDBC"：事务控制程序员处理

"MANAGED"：事务控制由容器管理(Spring)

--\>

\<transactionManager type="JDBC"/\>

\<!--

配置数据源

type：指定配置方式

"JNDI"：java命名目录接口，在服务器端进行数据库连接池的管理

"POOLED"：使用数据库连接池

"UNPOOLED"：不使用数据库连接池

--\>

\<dataSource type="POOLED"\>

\<property name="driver" value="\${jdbc.driver}"/\>

\<property name="url" value="\${jdbc.url}"/\>

\<property name="username" value="\${jdbc.username}"/\>

\<property name="password" value="\${jdbc.password}"/\>

\</dataSource\>

\</environment\>

\</environments\>
5.注册xxxMapper.xml文件
\<!--
resource:从resources目录下找指定名称的文件注册

url:使用绝对路径注册

class:动态代理方式下的注册
--\>
\<mappers\>
\<!-- 单个xml注册--\>

\<!-- \<mapper class="com.dhrj.zs.mapper.CustomerMapper"/\>--\>

\<package name="com......mapper"/\>
\</mappers\>

