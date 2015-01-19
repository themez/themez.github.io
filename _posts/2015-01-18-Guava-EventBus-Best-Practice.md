---
layout: post
title: 使用Guava EventBus的一些最佳实践
category: tech
tags: [Guava]
---
    
Guava的EventBus是一个基于订阅的消息总线库。在某些业务场景我们用它来在单进程中替代RabbitMQ来做事件分发。
它可以实现类似MQ类似的事件模型，但由于没有持久化，消息堆积能力取决于内存大小和线程池任务队列的配置。以下几点是在过去的使用中总结出来的一些最佳实践， 一般情况下只使用AsyncEventBus，除非必须要同步的场景：

>**Glossary**：
Listener: 包含handler方法的对象，监听一种或多种类型的Event
Handler: 处理某种类型Event的方法，使用@Subscribe注解

 - 针对一个事件如果有多个任务，注册多个handler，使用不同的handler去处理相应的任务
 - handler里面尽量不要阻塞，以免过早消耗完线程池中的资源
 - handler 不要抛出异常，抛出的异常不会被处理，应该自己捕获异常自己处理
 - 利用继承来实现event的复用
 - 使用不同的event bus实例来隔离业务，关键业务使用专属的实例(即专属的线程池)
 

EventBus隔离：
创建一个抽象listener，其他的listener实现getEventBusToRegister方法后拥有自动注册的能力。

    public abstract class AbstractEventListener {
		@PostConstruct
		public void init(){
			for(EventBus eventBus: getRegisterBus()){
				eventBus.register(this);
			}
		}
		protected abstract Set<EventBus> getEventBusToRegister();
	}

针对某类别时间使用默认的EventBus

	public abstract class BasePaymentEventListener extends AbstractEventListener{
	    @Resource
	    private EventBus paymentEventBus;
	    
	    @Override
	    protected Set<EventBus> getEventBusToRegister() {
	        return ImmutableSet.of(paymentEventBus);
	    }
	}

全局统一的EventListener可以注册在所有EventBus上：

	@Component
	public class DeadEventHandler extends AbstractEventListener{
	    private final static Logger logger = LoggerFactory.getLogger(DeadEventHandler.class);
	    @Resource
	    private EventBus eventBus1;
	    @Resource
	    private EventBus eventBus2;
	    
	    @Subscribe
	    public void handlerDeadEvent(DeadEvent event){
	        //handle
	    }
	    @Override
	    protected Set<EventBus> getEventBusToRegister() {
	        return ImmutableSet.of(eventBus1, eventBus2);
	    }
	}

遇到比较庞大的任务，更好的做法可能是为每个业务创建单独的线程池，handler方法只需要往对应的线程池提交任务。
