---
layout: post
title: 三种实现JDBC连接池的方法
date: 2018-12-29
categories: blog
tags: [JDBCPOOL]
description: jdbc连接池
---

常见的3种连接池实现方法：

1.自定义

2.DBCP开源连接池

3.C3P0开源连接池

---

<h1>1.自定义一个连接池池类</h1>


1.创建一个MyPool类：

声明： 一个LinkedList 链表存储连接对象；

声明：int initSize/maxSize 属性，记录连接池初始数量和最大数量

声明：int currentSize 属性，记录当前可用连接数

2.使用静态块，在类加载的时候自动注册驱动

3.定义一个无参构造函数，在构造时，初始化存储链表，存入initSize指定数量的可用连接

4.定义一个 getConnect（） 方法，用以获取数据库连接对象，考虑到用户/线程使用完一个连接后，习惯性的调用close 方法关闭连接，无法做到重复利用，固在获取数据库连接时，采用动态代理方法，重写Connection接口的close方法，具体代码实现，请参照下方 MyPool类中getCon（）方法

5.分别定义 从存储链表中获取连接的 getConnectFromPool() 方法 和 使用完毕释放连接的 releaseConnect() 方法

	public class MyPool {
	    private static String driver="com.mysql.jdbc.Driver";
	    private static String url="jdbc:mysql://localhost:3306/studio?serverTimezone=UTC&amp&characterEncoding=utf-8";
	    private static String userName="root";
	    private static String password="898218";
	    private LinkedList<Connection> conList=new LinkedList<>();
	    private int initSize=3;
	    private int maxSize=5;
	    private int currentSize=0;//记录当前可用连接数
	
	    //加载的时候自动注册驱动
	    static {
	        try {
	            Class.forName(driver);
	        } catch (ClassNotFoundException e) {
	            e.printStackTrace();
	        }
	    }
	    public MyPool(){
	        for(int i=0;i<initSize;i++){
	            conList.add(this.getCon());
	            currentSize++;
	        }
	    }
	    /**
	     * 获得连接方法
	     * @return
	     */
	    private Connection getCon(){
	            try {
	                final Connection con=DriverManager.getConnection(url,userName,password);
	                //动态代理，重写close方法
	                Connection myCon=(Connection) Proxy.newProxyInstance(
	                        MyPool.class.getClassLoader(),//类加载器，只要在同一个JDK中的类即可
	                        new Class[]{Connection.class},//要代理的接口的集合
	                        //编写一个方法处理器
	                        new InvocationHandler(){
	                            @Override
	                            public Object invoke(Object proxy, Method method, Object[] args)
	                                    throws Throwable {
	                                Object value=null;
	                                //当遇到close方法，就会把对象放回连接池中，而不是关闭连接
	                                if(method.getName().equals("close")){
	                                    conList.addLast(con);
	                                }else{
	                                    value=method.invoke(con,args);
	                                }
	                                return value;
	                            }
	                        }
	                );
	                return myCon;
	            } catch (SQLException e) {
	                e.printStackTrace();
	                throw new RuntimeException(e);
	            }
	    }
	    /**
	     * 从连接池中获取连接
	     */
	    public Connection getConFormPool(){
	        Connection connection=null;
	        if(conList.size()>0){
	            connection=conList.removeFirst();
	        }else if(conList.size()==0&&currentSize<maxSize){
	            //连接池被拿空，且连接数没有达到上限，创建新的连接
	            conList.addLast(this.getCon());
	            currentSize++;
	            connection=conList.removeFirst();
	        }else{
	            System.out.println("抱歉！");
	        }
	
	        return connection;
	    }
	    /**
	     * 释放连接
	     * @param con
	     */
	    public void releaseCon(Connection con){
	        conList.addLast(con);
	    }
	}


以上是自定义的一个简单的连接池类，已经具备了一个连接池的基础功能，还可以在此基础上继续增加更多自定义功能。


<h1>2.使用DBCP开源连接池</h1>

<h4>DBCP连接池是开源组织Apache软件基金组织开发的连接池实现。事实上，tomcat服务器默认就会使用这个连接池道具。</h4>

使用步骤：

1.下载DBCPjar包，添加到项目中

2.读取数据库连接配置信息：driver/url/userName/password

3.创建 BasicDataSource 连接池对象，传入配置信息

4.需要使用连接时，通过创建的连接池对象，调用 getConnection()方法，获取数据库连接对象，用完需要释放时，直接 close 即可
	
	public class DBCPTest {
	    private static String driver="com.mysql.jdbc.Driver";
	    private static String userName="root";
	    private static String password="898218";
	    public static void main(String[] args){
	        //获得连接池对象
	        BasicDataSource bs=new BasicDataSource();
	        bs.setUrl("jdbc:mysql://localhost:3306/studio?serverTimezone=UTC&amp&characterEncoding=utf-8");
	        bs.setUsername(userName);
	        bs.setPassword(password);
	        bs.setDriverClassName(driver);
	
	        bs.setInitialSize(3);
	        bs.setMaxActive(5);
	        bs.setMaxWait(1000);//设置最大等待时间，单位毫秒
	        //待到结束都没有连接被释放回连接池，就会出现报错。
	        for(int i = 0 ; i<9;i++)
	        {
	            Connection conn = null;
	            try {
	                conn = bs.getConnection();
	                System.out.println(conn.hashCode());
	                if(i==5)
	                {
	                    conn.close();
	                }
	            } catch (SQLException e) {
	                e.printStackTrace();
	            }
	        }
	    }

	}


<h1>3.使用C3P0开源数据连接池<h1>

<h4>C3P0是一个开源组织的产品，开源框架的内部的连接池一般都使用C3P0来实现，例如：Hibernate</h4>

1.C3P0的用法和DBCP的用法非常的相似~照搬上面的用例即可

2.特别的是C3PO读取参数文件的方式，C3P0除了能像DBCP那样读取配置文件，它还提供了一种特殊的设置参数的方式，就是把参数数据写在一个名叫c3p0-config.xml的XML文件中，在创建C3P0对象时会自动在classpath去寻找该文件来读取~
 ***也就是说:c3p0会到classpath下读取名字为c3p0-config.xml文件***
 

	public class C3P0Test {
	    private static String driver="com.mysql.jdbc.Driver";
	    private static String url="jdbc:mysql://localhost:3306/studio?serverTimezone=UTC&amp&characterEncoding=utf-8";
	    private static String userName="root";
	    private static String password="898218";
	    public static void main(String[] args){
	        ComboPooledDataSource cds=new ComboPooledDataSource();
	        //设置参数
	        cds.setJdbcUrl(url);
	        cds.setUser(userName);
	        cds.setPassword(password);
	        try {
	            cds.setDriverClass(driver);
	        } catch (PropertyVetoException e) {
	            e.printStackTrace();
	        }
	        //设置连接池的参数
	        cds.setInitialPoolSize(3);
	        cds.setMaxPoolSize(5);
	        cds.setCheckoutTimeout(1000);//最大等待时间
	        for(int i = 0 ; i<8 ; i++)
	        {
	            Connection conn = null;
	            try {
	                conn = cds.getConnection();
	            } catch (SQLException e) {
	                e.printStackTrace();
	            }
	            System.out.println(conn.hashCode());
	        }
		}
	}
