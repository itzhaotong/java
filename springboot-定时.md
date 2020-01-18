# SpringBoot 自带定时框架(企业微信推送操作)

## 1----->初始化线程池

```
package com.cs.qywx.task;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
@Configuration
public class TaskConfig {

	@Bean
	public ThreadPoolTaskScheduler trPoolTaskScheduler(){
		ThreadPoolTaskScheduler threadPoolTaskScheduler = new ThreadPoolTaskScheduler();
		threadPoolTaskScheduler.setPoolSize(20);
		threadPoolTaskScheduler.setThreadNamePrefix("taskExecutor-");
		threadPoolTaskScheduler.setWaitForTasksToCompleteOnShutdown(true);
		threadPoolTaskScheduler.setAwaitTerminationSeconds(60);
		return threadPoolTaskScheduler;
	} 
}
```



## 2 ----->构建定时计划参数

1>定时执行业务逻辑(属性可自己调整)

```
public class MyRunable implements Runnable{
	
	public String depid;//部门id
	public String ids;//个人唯一标识
	public String textContent;//推送内容
	public String status;//1:单人推送 2部门推送 ',
	
	get set ....
	
	@Override
	public void run() {
		System.out.println("定时任务执行中:" + DateUtil.fmDate(new Date(), 			DateUtil.long_date_format));
		业务逻辑代码....
		
	}
```

2>定时任务管理类

```
public class Timing implements Serializable{
	/**
	 * 定时任务管理类
	 */
	private static final long serialVersionUID = 1L;
	
	private Integer id;
	/**
	 * 部门id
	 */
    private Integer depid;
    /**
     * 推送内容主键id
     */
    private Integer sendRecordId;
    /**
     * 管理员id
     */
    private Integer adminId;
    /**
     * 成员id
     */
    private String userid;
    /**
     * 定时表达式
     */
    private String cron;
    /**
     * 1开启 0关闭',
     */
    private Integer type;
    /**
     * 1:单人推送 2部门推送 ',
     */
    private String status;
    /**
     * 线程名 
     */
    private String threedName;
    /**
     * 创建时间
     */
    private Date createTime;
    /**
     * 修改时间
     */
    private Date updateTime;
    
    ...get set方法
}
```



## 3 ----->实现开启定时任务



```
@Service
@EnableScheduling
public class DemoServiceImpl implements DemoService {

	@Autowired private DemoMapper demoMapper;
	@Autowired private ThreadPoolTaskScheduler threadPoolTaskScheduler;
	private Map<String,ScheduledFuture<?>> future = new ConcurrentHashMap<>();
	
	@Override
	public Result stopTask(Integer timingId) {
		try {
			Timing timing = timingMapper.selectByPrimaryKey(timingId);
			if(timing.getType() == 0){
				return Result.successReuslt("任务已暂停", null);
			}
			ScheduledFuture<?> scheduledFuture = future.get(timing.getThreedName());
			if(scheduledFuture!=null){
				scheduledFuture.cancel(true);
			}
			timing.setType(0);
			timingMapper.updateByPrimaryKeySelective(timing);
			System.out.println(timing.getThreedName() + "已任务暂停");
		} catch (Exception e) {
			e.printStackTrace();
			return Result.erroReuslt("任务异常!");
		}
		
		return Result.successReuslt("暂停成功", null);
	}



	@Override
	public Result startTask(Integer timingId) {
		try {
			/**
			 * task:定时任务要执行的方法
			 * trigger:定时任务执行的时间
			 */
			Timing timing = timingMapper.selectByPrimaryKey(timingId);
			if(timing.getType() == 1){
				return Result.successReuslt("正在进行中", null);
			}
			SendRecord srd = sendRecordMapper.selectByPrimaryKey(timing.getSendRecordId());
			MyRunable rb = new MyRunable(timing.getDepid().toString(), timing.getUserid(), srd.getContent(), timing.getStatus());
			CronTrigger ctr = new CronTrigger(timing.getCron());
			ScheduledFuture<?> futrues =  threadPoolTaskScheduler.schedule(rb,ctr);
			future.put(timing.getThreedName(), futrues);
			timing.setType(1);
			timingMapper.updateByPrimaryKeySelective(timing);
		} catch (Exception e) {
			e.printStackTrace();
			return Result.erroReuslt("推送服务异常");
		}
		
		return Result.successReuslt("开始执行", null);
	}



	@Override
	public Result deleTask(Integer timingId) {
		try {
			Timing timing = timingMapper.selectByPrimaryKey(timingId);
			if(timing == null){
				return Result.erroReuslt("任务不存在");
			}
			ScheduledFuture<?> scheduledFuture = future.get(timing.getThreedName());
			if(scheduledFuture!=null){
				scheduledFuture.cancel(true);
			}
			future.remove(timing.getThreedName());
			timingMapper.deleteByPrimaryKey(timingId);
		} catch (Exception e) {
			e.printStackTrace();
			return Result.erroReuslt("任务异常!");
		}
		return Result.successReuslt("删除成功!", null);
		
	}
}
```





































