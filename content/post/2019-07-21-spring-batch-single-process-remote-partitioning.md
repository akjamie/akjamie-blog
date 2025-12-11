---
layout: post
title: "Spring batch - single process & remote partitioning(multi instances)"
date: 2019-07-23 13:36:10
subtitle: ""
excerpt: ""
description: "spring batch 批处理poc"
author: "Jamie Zhang"
image: "/img/background-03.jpg"
tags:
     - Spring Batch
categories: [ Microservice ]
---
Spring batch是轻量级，全面的批处理框架，旨在开发对企业系统日常运营至关重要的强大批处理应用程序。

Spring Batch负责大量记录，包括日志记录/跟踪，事务管理，作业处理统计，作业重启，跳过和资源管理。 它还提供更高级的技术服务和功能，通过优化和分区技术实现极高容量和高性能的批处理作业。 简单和复杂的大批量批处理作业可以高度可扩展的方式利用框架来处理大量信息。

关于spring batch的细节介绍，请参加[Spring batch](https://spring.io/projects/spring-batch)官网，本文主要是记录一个poc的过程，主要包含单个实例应用和远程分析多实例处理的cases。

模拟demo的用户场景：  
定时处理多个系统输出的csv文件，然后跑批处理到数据到DB，简要架构图如下:  
![](/img/2019-07-21-spring-batch-single-process-remote-partitioning/20190721193530.png)

# 单个实例处理(且单个文件)
通过cron job定时调度batch job，整个job由两个step组成的一个flow，step1 - 清理目标数据库表(模拟场景，可能是数据备份，建立新的day表等); step2 - 加载文件，解析数据内容并转化为java对象,经processor处理，后倒入db存储.
sequence diagram:  
![](/img/2019-07-21-spring-batch-single-process-remote-partitioning/single-process-sequence.png)
主要配置文件  
## SingleProcessConfiguration
```
package org.akj.batch.configuration;

import lombok.extern.slf4j.Slf4j;
import org.akj.batch.constant.Constant;
import org.akj.batch.entity.Person;
import org.akj.batch.listener.CustomJobExcecutionListener;
import org.akj.batch.processor.PersonItemProcessor;
import org.akj.batch.repository.PeopleRepository;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.StepContribution;
import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.scope.context.ChunkContext;
import org.springframework.batch.core.step.tasklet.Tasklet;
import org.springframework.batch.item.data.RepositoryItemWriter;
import org.springframework.batch.item.data.builder.RepositoryItemWriterBuilder;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.builder.FlatFileItemReaderBuilder;
import org.springframework.batch.item.file.mapping.BeanWrapperFieldSetMapper;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.beans.propertyeditors.CustomDateEditor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;
import org.springframework.core.task.SimpleAsyncTaskExecutor;
import org.springframework.core.task.TaskExecutor;
import org.springframework.jdbc.core.JdbcTemplate;

import java.beans.PropertyEditor;
import java.io.File;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableBatchProcessing
@Slf4j
public class SingleSourceConfiguration {

    @Autowired
    public JobBuilderFactory jobBuilderFactory;

    @Autowired
    public StepBuilderFactory stepBuilderFactory;

    @Autowired
    private PeopleRepository repository;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Value("${batch.job.name}")
    private String jobName;

    @Value("${batch.job.chunk.size}")
    private int chunkSize;

    @Value("${batch.job.input.path}")
    private String inputFolder;

    @Bean
    public FlatFileItemReader<Person> reader() {
        BeanWrapperFieldSetMapper<Person> beanMapper = new BeanWrapperFieldSetMapper<Person>();
        beanMapper.setTargetType(Person.class);

        Map<String, PropertyEditor> customEditors = new HashMap<String, PropertyEditor>();
        DateFormat dateFormatter = new SimpleDateFormat("yyyy-MM-dd");
        customEditors.put("java.util.Date", new CustomDateEditor(dateFormatter, false));
        beanMapper.setCustomEditors(customEditors);
        return new FlatFileItemReaderBuilder<Person>().name("personItemReader")
                .resource(new FileSystemResource(inputFolder + File.separatorChar + Constant.FILE_SOURCE)).delimited()
                .names(new String[]{"pid", "firstName", "lastName", "age", "gender", "dateOfBirth", "height",
                        "weight"})
                .fieldSetMapper(beanMapper).build();
    }

    @Bean
    public PersonItemProcessor processor() {
        return new PersonItemProcessor();
    }

    @Bean
    public RepositoryItemWriter<Person> writer() {
        return new RepositoryItemWriterBuilder<Person>().repository(repository).methodName("save").build();
    }

    @Bean
    public Job importUserJob(CustomJobExcecutionListener listener, Step step1) {
        return jobBuilderFactory.get(jobName).incrementer(new RunIdIncrementer()).listener(listener).flow(step1).end()
                .build();
    }

    @Bean
    public Job importUserJobflow(CustomJobExcecutionListener listener, Step step1) {
        return jobBuilderFactory.get(jobName).listener(listener).start(step1()).on("COMPLETED").to(step2(writer()))
                .end().build();
    }

    @Bean
    public Step step1() {
        return stepBuilderFactory.get("step1").tasklet(new Tasklet() {
            @Override
            public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
                jdbcTemplate.execute(Constant.TRANCATE_TABLE_SQL);
                log.debug("successfully execute sql: " + Constant.TRANCATE_TABLE_SQL);
                return RepeatStatus.FINISHED;
            }
        }).build();

    }

    @Bean
    public Step step2(RepositoryItemWriter<Person> writer) {
        return stepBuilderFactory.get("step2").<Person, Person>chunk(chunkSize).reader(reader()).processor(processor())
                .writer(writer).taskExecutor(taskExecutor()).build();
    }

    @Bean
    public TaskExecutor taskExecutor() {
        return new SimpleAsyncTaskExecutor("spring batch");
    }
}
```

## ScheduledJobLancherConfiguration  
```
package org.akj.batch.configuration;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.JobParametersInvalidException;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.batch.core.launch.support.SimpleJobLauncher;
import org.springframework.batch.core.repository.JobExecutionAlreadyRunningException;
import org.springframework.batch.core.repository.JobInstanceAlreadyCompleteException;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.repository.JobRestartException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;

@EnableScheduling
@Configuration
@ConditionalOnProperty("batch.scheduling.enabled")
@Profile({"master"})
public class ScheduledJobLancherConfiguration {
    @Autowired
    Job job;

    @Value("${batch.job.name}")
    private String jobName;

    @Autowired
    private JobRepository repository;

    @Bean("scheduledJobLauncher")
    public JobLauncher jobLauncher(JobRepository jobRepo) {
        SimpleJobLauncher simpleJobLauncher = new SimpleJobLauncher();
        simpleJobLauncher.setJobRepository(jobRepo);
        return simpleJobLauncher;
    }

    @Scheduled(cron = "${batch.job.scheduler.cron}")
    public void perform() throws JobExecutionAlreadyRunningException, JobRestartException,
            JobInstanceAlreadyCompleteException, JobParametersInvalidException {
        JobParameters params = new JobParametersBuilder().addString(jobName, String.valueOf(System.currentTimeMillis()))
                .toJobParameters();
        jobLauncher(repository).run(job, params);
    }
}
```
配置文件 in application.yml
```
batch:
  scheduling:
    enabled: true
  partitioner:
    partition: 1 # partition count, grid size
  job:
    name: importUserDataTest
    chunk.size: 20
    input:
      path: c:/temp/source
      filter: '.csv'
    ouput:
      path: c:/temp/target
    failure:
      path: c:/temp/failure
    scheduler:
      cron: 0 */1 * * * ? #[s] [m] [h] [d] [m] [w] [y]
```
single process可以对应大部分场景了，并且可以通过设置taskExecutor来支持多线程处理  
```
@Bean
public TaskExecutor taskExecutor(){
    return new SimpleAsyncTaskExecutor("spring_batch");
}

@Bean
public Step sampleStep(TaskExecutor taskExecutor) {
        return this.stepBuilderFactory.get("sampleStep")
                                .<String, String>chunk(10)
                                .reader(itemReader())
                                .writer(itemWriter())
                                .taskExecutor(taskExecutor)
                                .build();
```

当数据量或者处理步骤更加复杂的场景时，JVM多线程都难以应对时，spring batch也提供了远程分块和远程分区支持，把task调度到远程instance上进行然后聚合，下面演示一个主从架构的多实例remote partition处理架构

# Remote partition主从架构
在single process的基础进行修改，保留处理流程step，并将其修改为master/slave step结构，主要流程为:  
1. cron task trigger jobLauncher 运行job任务
2. 运行master step任务分区并存储在DB，public Map<String, ExecutionContext> partition(int gridSize)
3. 通过AMQP协议，将partitioned task(StepExecution)分派到message queue
4. Slave node通过监听message queue，接受partioned task，运行slave step并更新运行结果
5. 所有step运行完成后，job complete.
架构如下:
![](/img/2019-07-21-spring-batch-single-process-remote-partitioning/remote-partition-diagram.png)

## Partitioner 分区器
```
package org.akj.batch.configuration.partitioner;

import lombok.extern.slf4j.Slf4j;
import org.akj.batch.util.PathMatchingResourcePatternResolver;
import org.springframework.batch.core.partition.support.Partitioner;
import org.springframework.batch.item.ExecutionContext;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.io.File;
import java.io.IOException;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

@Component
@Slf4j
public class CustomMultiResourcePartitioner implements Partitioner {

	@Value("${batch.job.input.path}")
	private String inputFolder;

	@Value("${batch.job.input.filter}")
	private String fileFilter;

	@Value("${batch.partitioner.partition}")
	private int partition;

	@Override
	public Map<String, ExecutionContext> partition(int gridSize) {
		Map<String, ExecutionContext> map = new HashMap<>(gridSize);
		AtomicInteger partitionNumber = new AtomicInteger(1);
		PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
		try {
			String[] resources = resolver.getResources(inputFolder, fileFilter);
			log.info("started partitioning, " + resources.length + " resources been found as input");
			Arrays.stream(resources).forEach(filePath -> {
					ExecutionContext context = new ExecutionContext();
					context.putString("fileName", filePath);
					map.put("partition" + partitionNumber.getAndIncrement(), context);
			});
		} catch (IOException e) {
			log.error("system error, invalid input resource: " + inputFolder + File.separatorChar + fileFilter);
		}

		return map;
	}
}

```
## AMQP配置
```
package org.akj.batch.configuration.partitioner;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.amqp.rabbit.AsyncRabbitTemplate;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.amqp.outbound.AsyncAmqpOutboundGateway;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.channel.NullChannel;
import org.springframework.integration.channel.QueueChannel;
import org.springframework.integration.config.EnableIntegration;
import org.springframework.integration.core.MessagingTemplate;
import org.springframework.messaging.PollableChannel;

@Configuration
@EnableIntegration
@Slf4j
public class RemotePartitioningAMQPConfiguration {

	@Bean
	@ServiceActivator(inputChannel = "outboundRequests")
	public AsyncAmqpOutboundGateway amqpOutboundEndpoint(AsyncRabbitTemplate asyncTemplate) {
		AsyncAmqpOutboundGateway gateway = new AsyncAmqpOutboundGateway(asyncTemplate);
		gateway.setExchangeName("user-service.batch");
		gateway.setRoutingKey("partition.requests");
		gateway.setRequiresReply(false);

		log.info("Created AMQP Outbound gateway---> " + gateway.toString());
		return gateway;
	}

	@Bean
	public AsyncRabbitTemplate asyncTemplate(RabbitTemplate rabbitTemplate) {
		return new AsyncRabbitTemplate(rabbitTemplate);
	}

	@Bean
	public QueueChannel inboundRequests() {
		return new QueueChannel();
	}

	@Bean
	public DirectChannel outboundRequests() {
		return new DirectChannel();
	}

	@Bean
	public PollableChannel outboundStaging() {
		return new NullChannel();
	}

	@Bean
	public Queue requestQueue() {
		return new Queue("partition.requests", true);
	}

	@Bean
	Binding binding(Queue queue, TopicExchange exchange) {
		return BindingBuilder.bind(queue).to(exchange).with("partition.requests");
	}

	@Bean
	public TopicExchange batch() {
		return new TopicExchange("user-service.batch", true, false);
	}

	@Bean
	public MessagingTemplate messageTemplate() {
		MessagingTemplate messagingTemplate = new MessagingTemplate(outboundRequests());
		messagingTemplate.setReceiveTimeout(200000l);
		return messagingTemplate;
	}
	
	 @Bean
    @Profile({"slave"})
    @ServiceActivator(inputChannel = "inboundRequests", outputChannel = "outboundStaging", async = "true", poller = {@Poller(fixedRate = "1000")})
    public StepExecutionRequestHandler stepExecutionRequestHandler() {
        StepExecutionRequestHandler stepExecutionRequestHandler = new StepExecutionRequestHandler();
        BeanFactoryStepLocator stepLocator = new BeanFactoryStepLocator();
        stepLocator.setBeanFactory(this.applicationContext);
        stepExecutionRequestHandler.setStepLocator(stepLocator);
        stepExecutionRequestHandler.setJobExplorer(this.jobExplorer);

        return stepExecutionRequestHandler;
    }

    @Bean
    @Profile({"slave"})
    public AmqpInboundChannelAdapter inbound(SimpleMessageListenerContainer listenerContainer) {
        AmqpInboundChannelAdapter adapter = new AmqpInboundChannelAdapter(listenerContainer);
        adapter.setOutputChannel(inboundRequests);
        adapter.afterPropertiesSet();

        return adapter;
    }

    @Bean
    public SimpleMessageListenerContainer container(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        container.setQueueNames("partition.requests");
        // container.setConcurrency("10");
        // container.setConcurrentConsumers(1);
        container.setErrorHandler(t -> {
            log.error("message listener error," + t.getMessage());
        });
        container.setAutoStartup(false);

        return container;
    }
}
```
demo中用profile进行了区分，可以主从方式，一主多从，暂时不支持mixed模式，主节点无法兼做从节点，理论上可以做到。  
spring.profiles.active=master 主节点，可以进行batch任务调度和任务分区和job状态监控  
spring.profiles.active-slace 从节点，从message queue中拉去任务并执行，更新任务完成情况到db  
源码 - https://github.com/cloud-poc/springbatch-userservice


参考资料:  
https://blog.51cto.com/13501268/2177746  
https://cloud.tencent.com/developer/article/1096337  
https://asardana.com/2018/01/01/scaling-spring-batch-on-aws-with-remote-partitioning/  
https://docs.spring.io/spring-integration/docs/5.1.6.RELEASE/reference/html/#amqp  
https://www.docs4dev.com/docs/zh/spring-batch/4.1.x/reference/spring-batch-integration.html  
