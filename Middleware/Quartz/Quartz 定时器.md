# Quartz 定时器

## 官网

- [官网地址](http://www.quartz-scheduler.org/)

- [官网教程地址](http://www.quartz-scheduler.org/documentation/quartz-2.3.0/tutorials/) 建议看一遍
- [github 地址](https://github.com/quartz-scheduler/quartz)

## 环境

- jar 包,当前稳定版本

  ```xml
  <dependency>
      <groupId>org.quartz-scheduler</groupId>
      <artifactId>quartz</artifactId>
      <version>2.3.0</version>
  </dependency>
  ```

- quartz.properties 配置文件在jar 中有`\org\quartz\quartz.properties`，默认配置文件

  ```properties
  # Default Properties file for use by StdSchedulerFactory
  # to create a Quartz Scheduler Instance, if a different
  # properties file is not explicitly specified.
  #
  
  org.quartz.scheduler.instanceName: DefaultQuartzScheduler
  org.quartz.scheduler.rmi.export: false
  org.quartz.scheduler.rmi.proxy: false
  org.quartz.scheduler.wrapJobExecutionInUserTransaction: false
  
  org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
  org.quartz.threadPool.threadCount: 10
  org.quartz.threadPool.threadPriority: 5
  org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread: true
  
  org.quartz.jobStore.misfireThreshold: 60000
  
  org.quartz.jobStore.class: org.quartz.simpl.RAMJobStore
  ```

## 例子

- 新建一个要执行的 Job

  ```java
  public class HelloJob implements Job {
      @Override
      public void execute(JobExecutionContext context) throws JobExecutionException {
          System.out.println("滑板鞋 摩擦摩擦");
          // 证明每次调用都是新创建一个对象
          System.out.println(this.hashCode());
      }
  }
  ```

- 新建一个 工作实例 和 一个触发器 绑定到任务调度器中

  ```java
  public class QuartzTest {
      public static void main(String[] args) {
          try {
              Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
              // 要执行的任务
              JobDetail job = JobBuilder.newJob(HelloJob.class)
                      .withIdentity("helloJob", "test")
                      .build();
              // 执行的时间
              Trigger trigger = TriggerBuilder.newTrigger()
                      .withIdentity("every-second", "test")
                      .startNow()
                      .withSchedule(SimpleScheduleBuilder
                              .simpleSchedule()
                              .withIntervalInSeconds(2)
                              .repeatForever())
                      .build();
  			// 绑定到任务调度器中
              scheduler.scheduleJob(job, trigger);
              scheduler.start();
              Thread.sleep(10000);
              scheduler.shutdown();
          } catch (SchedulerException | InterruptedException e) {
              e.printStackTrace();
          }
      }
  }
  ```

- 执行结果：

  ```log
  滑板鞋 摩擦摩擦
  1680663854
  滑板鞋 摩擦摩擦
  229664176
  滑板鞋 摩擦摩擦
  1078879095
  滑板鞋 摩擦摩擦
  1190201569
  滑板鞋 摩擦摩擦
  1993026392
  滑板鞋 摩擦摩擦
  1042017060
  ```

> - HelloJob 每次调用都是生成新的对象，（官网中有说明）
> - 调度器一定是单例的

## 体系

- Scheduler - the main API for interacting with the scheduler.
- Job - an interface to be implemented by components that you wish to have executed by the scheduler.
- JobDetail - used to define instances of Jobs.
- Trigger - a component that defines the schedule upon which a given Job will be executed.
- JobBuilder - used to define/build JobDetail instances, which define instances of Jobs.
- TriggerBuilder - used to define/build Trigger instances

> [以上概念是官网copy的](http://www.quartz-scheduler.org/documentation/quartz-2.3.0/tutorials/tutorial-lesson-02.html)

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/quratz/Snipaste_2021-03-03_20-59-34.png)

<center>总体架构</center>

- `JobDetail` 

  - 包装了 Job ，而且`JobDetail` 可以携带一个 `JobDataMap`

- `Trigger`

  - `JobDetail` 跟 `Trigger` 是 1:N 的关系,解耦可用实现多个规则触发任务

  - 触发器种类

    | 子接口                   | 描述                     | 特点                                           |
    | ------------------------ | ------------------------ | ---------------------------------------------- |
    | SimpleTrigger            | 简单触发器               | 固定时刻或时间间隔，毫秒                       |
    | CalendarIntervalTrigger  | 基于日历的触发器         | 比简单触发器更多时间单位，支持非固定时间的触发 |
    | DailyTimeIntervalTrigger | 基于日期的触发器         | 每天的某个时间段                               |
    | CronTrigger              | 基于 Cron 表达式的触发器 | 功能强大                                       |

  - Calendar 排除规则基本用法 `scheduler.addCalendar()`

- `Scheduler`

  - 单例
  - 实现类是 `StdScheduler`
  - 三大类方法：（可以实现任务的动态调度）
    - 操作调度器本身，例如调度器的启动 start()、调度器的关闭 `shutdown()`
    - 操作 `Trigger`，例如 `pauseTriggers()`、`resumeTrigger()`
    - 操作 `Job`，例如 `scheduleJob()`、`unscheduleJob()`、`rescheduleJob()`

- `Listener`

  - `JobListener`
  - `TriggerListener`
  - `SchedulerListener`

- `JobStore`

  - 用来存储任务和触发器相关的信息，例如所有任务的名称、数量、状态等

  - 存储方式有两种，内存或者数据库

    - `RAMJobStore`，默认方式

    - `JDBCJobStore`

      - JobStoreTX：在独立的程序中使用，自己管理事务，不参与外部事务

      - JobStoreCMT：如果需要容器管理事务时，使用它

      - 数据库表的位置：`src\org\quartz\impl\jdbcjobstore`,11张

      - 配置：

        ```properties
        org.quartz.jobStore.class:org.quartz.impl.jdbcjobstore.JobStoreTX
        org.quartz.jobStore.driverDelegateClass:org.quartz.impl.jdbcjobstore.StdJDBCDelegate
        # 使用 quartz.properties，不使用默认配置
        org.quartz.jobStore.useProperties:true
        #数据库中 quartz 表的表名前缀
        org.quartz.jobStore.tablePrefix:QRTZ_
        org.quartz.jobStore.dataSource:myDS
        
        #配置数据源
        org.quartz.dataSource.myDS.driver:com.mysql.jdbc.Driver
        org.quartz.dataSource.myDS.URL:jdbc:mysql://localhost:3306/xx?useUnicode=true&characterEncoding=utf8
        org.quartz.dataSource.myDS.user:root
        org.quartz.dataSource.myDS.password:123456
        org.quartz.dataSource.myDS.validationQuery=select 0 from dual
        ```


#### 注解

- `@DisallowConcurrentExecution` 不允许同一时间运行两相同的 Job
- `@PersistJobDataAfterExecution` 允许修改 JobDataMap 的数据



#### 错过触发定时任务

- `@DisallowConcurrentExecution`  修饰的时候可能错过触发，即两次任务触发事件叠加了
- `org.quartz.threadPool.threadCount` 设置的过小，而且同一时间执行的任务数超过了线程池大小

## 动态调度

- 建立一张表，参考 JobDetail 的属性

  ```sql
  CREATE TABLE `sys_job` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `job_name` varchar(512) NOT NULL COMMENT '任务名称',
  `job_group` varchar(512) NOT NULL COMMENT '任务组名',
  `job_cron` varchar(512) NOT NULL COMMENT '时间表达式',
  `job_class_path` varchar(1024) NOT NULL COMMENT '类路径,全类型',
  `job_data_map` varchar(1024) DEFAULT NULL COMMENT '传递 map 参数',
  `job_status` int(2) NOT NULL COMMENT '状态:1 启用 0 停用',
  `job_describe` varchar(1024) DEFAULT NULL COMMENT '任务功能描述',
  PRIMARY KEY (`id`)
  ) ENGINE=InnoDB AUTO_INCREMENT=25 DEFAULT CHARSET=utf8;
  ```

- 利用 Scheduler 可以 新增、删除、启动、暂停和修改任务

- 容器启动与 Service 注入

  1. 如何读取任务信息？

     创建一个类，实现 CommandLineRunner 接口，实现 run 方法，从表中查出状态是启动的任务，然后构建。

  2. Spring Bean 如何注入到实现了 Job 接口的类中？

     - 原因：因为定时任务 Job 对象的实例化过程是在 Quartz 中进行的，而 Service Bean 是由
       Spring 容器管理的，Quartz 察觉不到 Service Bean 的存在，所以无法将 Service Bean
       装配到 Job 对象中。

     - 分析：Quartz 集成到 Spring 中，用到 `SchedulerFactoryBean`，其实现了 `InitializingBean` 方法，在唯一的方法 `afterPropertiesSet()`在 Bean 的属性初始化后调用。调度器用 `AdaptableJobFactory` 对 Job 对象进行实例化。所以，如果我们可以把这个 `JobFactory` 指定为我们自定义的工厂的话，就可以在 Job 实例化完成之后，把 Job纳入到 Spring 容器中管理

       1. 定义一个 `AdaptableJobFactory`，实现 `JobFactory` 接口，实现接口定义的 `newJob` 方法，在这里面返回 Job 实例

       2. 定义一个 `MyJobFactory`，继承 `AdaptableJobFactory`。使用 Spring 的 `AutowireCapableBeanFactory`，把 Job 实例注入到容器中

       3. 指定 Scheduler 的 JobFactory 为自定义的 JobFactory

          ```java
          @Component
          public class MyJobFactory extends AdaptableJobFactory {
              @Autowired
              private AutowireCapableBeanFactory capableBeanFactory;
          
              protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
                  Object jobInstance = super.createJobInstance(bundle);
                  capableBeanFactory.autowireBean(jobInstance);
          
                  return jobInstance;
              }
          }
          ```

- [其他实现方式](https://blog.csdn.net/weixin_43424932/article/details/105713034)