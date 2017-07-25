Hi


<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:int="http://www.springframework.org/schema/integration"
	xmlns:jpa="http://www.springframework.org/schema/data/jpa"
	xmlns:aop="http://www.springframework.org/schema/aop"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:int-jpa="http://www.springframework.org/schema/integration/jpa"
	xmlns:int-jms="http://www.springframework.org/schema/integration/jms"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.3.xsd
		http://www.springframework.org/schema/data/jpa http://www.springframework.org/schema/data/jpa/spring-jpa-1.8.xsd
		http://www.springframework.org/schema/integration/jms http://www.springframework.org/schema/integration/jms/spring-integration-jms-4.3.xsd
		http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.3.xsd
		http://www.springframework.org/schema/integration/jpa http://www.springframework.org/schema/integration/jpa/spring-integration-jpa-4.3.xsd
		http://www.springframework.org/schema/integration http://www.springframework.org/schema/integration/spring-integration-4.3.xsd">
	
	
	<int-jpa:inbound-channel-adapter id="pollNdtTable"
		channel="ndtListChannel" entity-manager-factory="entityManagerFactory"
		auto-startup="true" 
		jpa-query="select ndt from NdtNeftDebitTbl ndt where ndt.ndtStatus=1 ORDER BY ndt.dmLstupddt"
		expect-single-result="false" max-results="1000">
		<int:poller fixed-delay="1000" id="ndtPoller">
			<int:transactional propagation="REQUIRED"
				transaction-manager="transactionManager" />
		</int:poller>		
	</int-jpa:inbound-channel-adapter>
	
	<int:splitter id="ndtListSplitter" input-channel="ndtListChannel"
		output-channel="ndtEntityChannel" />
		
	<int:service-activator input-channel="ndtEntityChannel"
		output-channel="updatendtChannel" ref="testService" id="printNdtEntity"
		method="testService" requires-reply="true" />
	
	<int-jpa:updating-outbound-gateway request-channel="updatendtChannel" entity-class="com.tcs.messaging.entity.NdtNeftDebitTbl" 
		entity-manager-factory="entityManagerFactory" reply-channel="jmsQueueChannel" flush="true"> <int-jpa:transactional transaction-manager="transactionManager" 
		 /> </int-jpa:updating-outbound-gateway>
		 
   <int-jms:outbound-channel-adapter id="jmsQueueChannel"
		connection-factory="mqConnection" destination="incomingMessages"
		explicit-qos-enabled="true" delivery-persistent="true" />
		 
	

	<bean
		class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">

		<property name="location">
			<value>jdbc.properties</value>
		</property>
	</bean>


	<bean id="conPoolDS" class="com.mchange.v2.c3p0.ComboPooledDataSource"
		destroy-method="close">
		<property name="driverClass" value="${jdbc.driverClassName}" />
		<property name="jdbcUrl" value="${jdbc.url}" />
		<property name="user" value="${jdbc.username}" />
		<property name="password" value="${jdbc.password}" />
		<property name="initialPoolSize" value="${jdbc.initialPoolSize}"/>
		<property name="maxPoolSize" value="${jdbc.maxPoolSize}" />
		<property name="minPoolSize" value="${jdbc.minPoolSize}" />
		<property name="maxStatements" value="${jdbc.maxStatements}" />
		<property name="testConnectionOnCheckout" value="${jdbc.testConnectionOnCheckout}"></property>
	</bean>


	<bean id="entityManagerFactory"
		class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
		<property name="packagesToScan" value="com.tcs.messaging.entity" />
		<property name="dataSource" ref="conPoolDS" />
		<property name="jpaVendorAdapter">
			<bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter" />
		</property>
		<property name="jpaProperties">
			<props>
				<prop key="hibernate.dialect">
					org.hibernate.dialect.Oracle10gDialect
				</prop>
				<prop key="hibernate.max_fetch_depth">3</prop>
				<prop key="hibernate.jdbc.fetch_size">50</prop>
				<prop key="hibernate.jdbc.batch_size">10</prop>
				<prop key="hibernate.show_sql">true</prop>
			</props>
		</property>
	</bean>

	<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
		<property name="entityManagerFactory" ref="entityManagerFactory" />
	</bean>
	
	
	<aop:config>
		<aop:pointcut id="serviceOperation"
			expression="execution(* com.tcs.messaging.service.*.*(..))" />
		<aop:advisor pointcut-ref="serviceOperation" advice-ref="txAdvice" />
	</aop:config>
	
	<tx:advice id="txAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="find*" read-only="true" rollback-for="java.lang.Throwable" />
			<tx:method name="count*" propagation="NEVER" rollback-for="java.lang.Throwable" />
			<tx:method name="*" rollback-for="java.lang.Throwable" />
		</tx:attributes>
	</tx:advice>
	
	<jpa:repositories base-package="com.tcs.messaging.repository"
		entity-manager-factory-ref="entityManagerFactory"
		transaction-manager-ref="transactionManager" />
	<jpa:auditing />
	
	<bean id="mqConnection" class="com.ibm.mq.jms.MQConnectionFactory">
         <property name="queueManager" value="MESSAGEQMGR"/>		
	</bean>
	
	<bean id="incomingMessages" class="com.ibm.mq.jms.MQQueue">
		<constructor-arg value="INCOMINGMESSAGES" />
	</bean>


</beans>


jdbc.driverClassName=oracle.jdbc.driver.OracleDriver
#jdbc.url=jdbc:oracle:thin:@localhost:1928:test
jdbc.url=jdbc:oracle:thin:@xxxxxxxx:1521:
jdbc.username=
jdbc.password=
jdbc.initialPoolSize=12
jdbc.maxPoolSize=25
jdbc.minPoolSize=6
jdbc.maxStatements=50
jdbc.testConnectionOnCheckout=true


<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.tcs.messaging</groupId>
	<artifactId>spring-integration</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>spring-integration</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.4.RELEASE</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<c3p0.version>0.9.5.1</c3p0.version>
		
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-integration</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.integration</groupId>
			<artifactId>spring-integration-jpa</artifactId>
		</dependency>
		
		<dependency>
			<groupId>org.springframework.integration</groupId>
			<artifactId>spring-integration-jms</artifactId>
		</dependency>
		
		

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

		<!-- c3p0 connection pool -->
		<dependency>
			<groupId>com.mchange</groupId>
			<artifactId>c3p0</artifactId>
			<version>${c3p0.version}</version>
		</dependency>
		

	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>


</project>
