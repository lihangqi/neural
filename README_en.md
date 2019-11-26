[中文](/README.md) | English

# Neural

[TOC]

	<-. (`-')_  (`-')  _              (`-')  (`-')  _           
	   \( OO) ) ( OO).-/     .->   <-.(OO )  (OO ).-/    <-.    
	,--./ ,--/ (,------.,--.(,--.  ,------,) / ,---.   ,--. )   
	|   \ |  |  |  .---'|  | |(`-')|   /`. ' | \ /`.\  |  (`-') 
	|  . '|  |)(|  '--. |  | |(OO )|  |_.' | '-'|_.' | |  |OO ) 
	|  |\    |  |  .--' |  | | |  \|  .   .'(|  .-.  |(|  '__ | 
	|  | \   |  |  `---.\  '-'(_ .'|  |\  \  |  | |  | |     |' 
	`--'  `--'  `------' `-----'   `--' '--' `--' `--' `-----'  


The neural organization in the microservice architecture mainly provides three major blades for cluster fault tolerance for distributed architecture: current limiting, degrading, and fusing. It also provides SPI, filter, JWT, retry mechanism, and plug-in mechanism. In addition, there are many small black technologies (such as IP black and white list, UUID enhanced version, Snowflake and large concurrent timestamp acquisition, etc.).

**Core functions**:
- **Limited flow**: Committed to solving the impact pressure of external flow
- **downgrade**: Committed to solving internal service failure events
- **Fuse**: Committed to the stability of internal services
- **Retry**: Committed to improving the success rate of external services

**Features**

- Distributed current limit (`Limiter`)
- Dedicated to the flow control of distributed service calls, you can limit the flow between the service calls and the service gateway!
- Service downgrade (`Degrade`)
- Committed to providing distributed service downgrade switches!
- Personalized retry (`Retryer`)
- Committed to building a smarter retry mechanism to show you the retest AI!
- Service authentication (`Auth`)
- Committed to ensuring the authentication of each distributed call, service authentication in the service registration, subscription and invocation links!
- Link Tracking (`Trace`)
    - Committed to providing link tracking for microservice architecture!
- Black Technology
    - `Perf`: a performance test artifact that can be used to test performance for a single method or block of code
    - `NUUID`: UUID extended version, providing richer UUID production rules
    - `Filter`: filter based on responsibility chain mode
    - `IPFilter`: IP black and white list filter
    - `Snowflake`: Distributed ID Generator based on Snowflake algorithm
    - `SystemClock`: Resolve performance issues when getting timestamps in large concurrent scenarios

## 1 SPI
### 1.1 SPI defects in JDK

- The JDK standard SPI will instantiate all implementations of the extension point at one time. It is time consuming to implement the extension implementation, but it will be a waste of resources if it is not loaded.
- IoC and AOP for extension points are not supported
- Do not support sorting
- Implementation class grouping is not supported
- Does not support single/multiple choices

### 1.2 SPI Features

- Support for custom implementation classes as singleton/multiple cases
- Support for setting the default implementation class
- Support for class order sorting
- Support implementation class definition feature attribute category, used to distinguish different categories of multiple dimensions
- Support for searching implementation classes based on category attribute values
- Support automatic scanning implementation class
- Support for manually adding implementation classes
- Support for all implementation classes
- Supports only creating the required implementation classes, solving the JDK native full way
- Support custom ClassLoader to load class

**TODO**: Support for extension point IoC and AOP needs to be implemented. One extension point can directly sink into other extension points.


### 1.3 How to use

**Step 1**: Define the interface
```java
@SPI
Public interface IDemo {}
```

**Step 2**: Define the interface implementation class
```java
@Extension("demo1")
Public class Demo1Impl implements IDemo {}

@Extension("demo2")
Public class Demo2Impl implements IDemo {}
```

**Step 3**: Create an interface resource file using the interface full path (package name class name)

`src/main/resources/META-INF/neural/io.neural.demo.IDemo`

**Step 4**: Write the implementation class full path (package name class name) in the interface resource file
```
io.neural.demo.Demo1Impl
io.neural.demo.Demo2Impl
```

**Step 5**: Use ExtensionLoader to get the interface implementation class
```java
Public class Demo{
    Public static void main(String[] args){
        IDemo demo1 = ExtensionLoader.getLoader(IDemo.class).getExtension("demo1");
        IDemo demo2 =ExtensionLoader.getLoader(IDemo.class).getExtension("demo2");
    }
}
```


## 2 Current Limit (Limiter)
In the distributed architecture, the traffic limiting scenarios are mainly divided into two types: injvm mode and cluster mode.

![redis-lua.png](docs/redis-lua.png)

### 2.1 injvm mode
#### 2.1.1 Concurrency
Use the semaphore in the JDK for control.

```java
Public class Test{
    Public static void main(String[] args){
        Semaphore semaphore = new Semaphore(10,true);
        Semaphore.acquire();
        //do something here
        Semaphore.release();
    }
}
```

#### 2.1.2 Rate Control (Rate)
Use the speed limiter (RateLimiter) in Google's Guava for control.

```java
Public class Test{
    Public static void main(String[] args){
        RateLimiter limiter = RateLimiter.create(10.0); // No more than 10 tasks per second are submitted
        Limiter.acquire(); // Request RateLimiter
    }
}
```

### 2.2 cluster mode (to be completed)
Distributed current limiting is mainly used to protect the security of the cluster or to strictly control the amount of requests from users (API economy).

Https://www.jianshu.com/p/a3d068f2586d

### 2.3 Limiting the number of instantaneous concurrency

- **Definition**: Instantaneous concurrency, the number of requests/transactions processed by the system simultaneously
- **Benefits**: This algorithm can achieve the effect of controlling the number of concurrent
- **Disadvantages**: The usage scenario is relatively simple and is generally used to control incoming traffic.

### 2.4 Limit time window maximum request number

- **Define**: Maximum number of requests for time window, maximum number of requests allowed in the specified time range
- **Advantages**: This algorithm can satisfy most of the flow control requirements. The maximum QPS can be directly converted by the maximum number of requests in the time window (QPS = Requests/Time Window)
- **Disadvantages**: This method may cause unsmooth traffic, and a small amount of traffic in the time window is particularly large.

### 2.5 Token bucket

**Algorithm Description**

- If the average sending rate configured by the user is r, one token is added to the bucket every 1/r second.
- Assume that up to b tokens can be stored in the bucket. If the token bucket is full when the token arrives, the token will be discarded.
- When traffic enters at rate v, the token is taken from the bucket at rate v, the traffic of the token is passed, and the token traffic is not passed, and the fuse logic is executed.

**Attributes**

- In the long run, the rate of compliance with traffic is affected by the token addition rate and is stabilized as: r
- Because the token bucket has a certain amount of storage, it can withstand certain traffic bursts.
    - M is the maximum possible transfer rate in bytes per second. M>r
    - T max = b/(M-r) Time to withstand the maximum transmission rate
    - B max = T max * M The amount of traffic transmitted during the time that is subject to the maximum transmission rate

**Advantages**: The traffic is smooth and can withstand certain traffic bursts


## 3 CircuitBreaker
In the distributed architecture, there are two main types of blown scenes: injvm mode and cluster mode.

### 3.1 Event Statistics Fuse (EventCountCircuitBreaker)
A lite fuse is implemented based on the number of times the event occurred during a specified time period. If 5 events are triggered within 10 seconds, the fuse is blown.

### 3.2 Threshold Circuit Breaker (ThresholdCircuitBreaker)
TODO


## 4 Degrade (to be completed)
Service downgrade refers to the downgrading of some services and pages according to the current business situation and traffic when the server pressure increases sharply, thereby alleviating the pressure on the server resources to ensure the normal operation of the core tasks, while ensuring some or even large Some customers get the right response.

### 4.1 Management method
#### 4.1.1 Direct management mode: The operation and maintenance personnel can specify which modules are downgraded.
When the server detects an increase in pressure, the server monitors automatically sends a notification to the operation and maintenance personnel. The operation and maintenance personnel degrade the system according to the judgment of the user or the relevant personnel and set the current operation level through the configuration platform. Downgrading can first downgrade interfaces to non-core services. If the effect is not significant, start downgrading some pages to ensure that the core functions are working properly.

#### 4.1.2 Hierarchical management mode: O&M personnel do not need to care about the details of the business, but can directly lower the level.
The service determines the priority level of the corresponding business and specifies the hierarchical degradation plan. When the server detects an increase in pressure, the service detection automatically sends a notification to the operation and maintenance personnel. The operation and maintenance personnel select the operation level according to the situation.


## 5 Retryer
### 5.1 Retry strategy
#### 5.1.1 Block Strategy (BlockStrategy)
Causes the current thread to perform a sleep retry using Thread.sleep().

#### 5.1.2 Stop Strategy (StopStrategy)

- **NeverStopStrategy**: Never stop strategy
- **StopAfterAttemptStrategy**: Stop the strategy after trying
- **StopAfterDelayStrategy**: Stop strategy after delay

#### 5.1.3 Waiting Strategy (WaitStrategy)

- **FixedWaitStrategy**: Fixed sleep time wait strategy
- **RandomWaitStrategy**: Random sleep time waiting strategy, support setting the lower limit (minmum) and upper limit (maxmum) of random sleep time
- **IncrementingWaitStrategy**: Fixed length incremental sleep time wait strategy
- **ExponentialWaitStrategy**: The exponential function (2^x, where x represents the number of attempts) increments the sleep time wait strategy. Support to set the upper limit of sleep time (maximumWait)
- **FibonacciWaitStrategy**: Fibonacci sequence increments sleep time wait strategy. Support to set the upper limit of sleep time (maximumWait)
- **CompositeWaitStrategy**: Composite wait policy, which supports the combination of the above waiting strategies to calculate the sleep time, and the final sleep time is the sum of the sleep times in the above strategy.
- **ExceptionWaitStrategy**: Exception Waiting Policy