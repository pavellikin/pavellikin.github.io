---
layout: post
title:  "Динамический Scheduling в spring boot с помощью триггеров"
date:   2017-12-24 17:00:34 +0300
categories: rus java spring-boot
---
Spring boot предоставляет удобную возможность по управлению задачами, которые должны быть выполнены через заданные промежутки времени. Для этого достаточно воспользоваться аннотацией @Scheduled.

Данная аннотация обладает следующими свойствами:
1. Запуск задачи по cron.
2. Запуск задачи с фиксированной задержкой (fixedDelay) между последним вызовом и началом следующего.
3. Запуск задачи с фиксированной задержкой (fixedRate) между вызовами.
4. Установка начальной (initialDelay) задержки перед выполнением задачи в миллисекундах.

Аннотацию @Scheduled достаточно поместить над методом, который должен выполняться периодически. Создадим класс SimpleScheduledTask, чтобы проиллюстрировать это.
``` java
@Component
public class SimpleScheduledTask implements ScheduledTask {
    @Scheduled(fixedDelay = 5000)
    @Override
    public void process() {
        System.out.println("Process...");
    }
}
```
Чтобы аннотация @Scheduled имела эффект во время выполнения, необходимо добавить аннотацию @EnableScheduling в вашу spring boot конфигурацию.
``` java
@Configuration
@EnableScheduling
public class SchedulingConfiguration {
}
```
При этом main класс будет выглядеть следующим образом.
``` java
@SpringBootApplication(scanBasePackages = "springbootexample")
public class SpringBootExampleApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootExampleApplication.class, args);
	}
}
```
Запустим приложение и убедимся, что задача выполняется.
```
2017-12-23 16:14:27.570  INFO 56251 --- [           main] e.s.SpringBootExampleApplication         : Started SpringBootExampleApplication in 1.571 seconds (JVM running for 2.012)
Process...
Process...
Process...
```
Если задача должна выполняться через динамически определяемые интервалы времени и потоком исполнения должен управлять spring, то на помощь нам придет сущность Trigger, а именно DynamicPeriodicTrigger. Для работы с DynamicPeriodicTrigger необходимо добавить зависимость на spring-integration-core.

Объявим новый класс DynamicScheduledTask, который будет содержать в себе ссылку на свой триггер, период исполнения по умолчанию, диапазон значений для генератора псевдослучайных чисел и сам генератор, который будет влиять на следующий интервал времени.
``` java
public class DynamicScheduledTask implements ScheduledTask {

    private final DynamicPeriodicTrigger trigger;
    private final int period;
    private final int range;
    private final Random generator;

    public DynamicScheduledTask(final DynamicPeriodicTrigger trigger, int period, int range) {
        this.trigger = trigger;
        this.period = period;
        this.range = range;
        this.generator = new Random();
    }

    @Override
    public void process() {
        int next = generator.nextInt(range);
        if (next % 2 == 0) {
            System.out.println(String.format("Set trigger to %d seconds", period));
            trigger.setPeriod(period);
        } else {
            System.out.println("Set trigger to 3 seconds");
            trigger.setPeriod(3);
        }
    }

    public DynamicPeriodicTrigger getTrigger() {
        return trigger;
    }
}
```
Теперь необходимо обновить SchedulingConfiguration класс - мы должны вручную создать bean для DynamicScheduledTask и связать его с DynamicPeriodicTrigger, через который будем управлять временным интервалом.
Кроме того необходимо зарегистрировать DynamicScheduledTask в spring.
``` java
@Configuration
@EnableScheduling
public class SchedulingConfiguration implements SchedulingConfigurer {

    @Value("${scheduling.period-in-seconds}")
    private int period;

    @Value("${scheduling.range}")
    private int range;

    @Bean
    public ScheduledTask dynamicScheduledTask() {
        DynamicPeriodicTrigger trigger = new DynamicPeriodicTrigger(period, TimeUnit.SECONDS);
        return new DynamicScheduledTask(trigger, period, range);
    }

    @Override
    public void configureTasks(final ScheduledTaskRegistrar taskRegistrar) {
        ScheduledTask task = dynamicScheduledTask();
        taskRegistrar.addTriggerTask(task::process, ((DynamicScheduledTask) task).getTrigger());
    }
}
```
Запустим приложение и убедимся, что управление триггером позволяет изменять интервалы запуска во время выполнения. 
```
2017-12-23 16:18:36.970  INFO 56251 --- [           main] e.s.SpringBootExampleApplication         : Started SpringBootExampleApplication in 1.571 seconds (JVM running for 2.012)
Set trigger to 5 seconds
Set trigger to 3 seconds
Set trigger to 5 seconds
``` 
Полный код приложения доступен по [ссылке](https://github.com/burnout171/spring-boot-example).

Источники:
1. <https://spring.io/guides/gs/scheduling-tasks/>
2. <https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html>