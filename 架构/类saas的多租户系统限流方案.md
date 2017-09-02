###一、问题起因
前几天在面试的时候，因为我以前有个B2B订货平台（saas系统架构，平台给每个租户，提供完全独立的在线商城服务，而所有的商城实际上还是在同一个系统中）的项目经验，所以面试官问到了这个问题，当某个租户的流量特别大，怎么保证其他租户的访问不受限制？

###二、一般系统常见的限流方案
####1.限制并发数
假设系统瞬时可接受的并发数为1000，那么每来一个请求将该数值减1，请求执行完成后将该数值加1，当该数值小于等于0时拒绝访问。该方法实现非常简单粗暴，java应用中可以通过java.util.corrurnent包下的Semaphore信号量的tryAcquire和release操作来实现。
该方法其实常见于一些长连接的限制上，比如db连接。对于执行时间较短，波动较大的请求，并不能很公平的限制流量，因为每个请求执行时间不一样，甚至不同系统负载下的同一个请求执行时间也不一样，如果都只获取一个并发数并不是一个很优的方案。（这个也是我当时面试的时候给面试官的方案，给每个租户一个固定并发数来限制，现在想来其实还有些欠妥）
####2.漏桶（Leaky Bucket）算法

![漏桶算法](https://upload-images.jianshu.io/upload_images/3298892-50b7a56d31ac079f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

初始一个定长的漏桶，将请求任务放到漏桶中，以固定的流出速率去执行请求，如果桶满了就丢弃。实现方式，事先准备一个定长队列，存放请求任务，准备一个任务执行线程池固定时间间隔取任务来执行即可，队列满了就丢弃。
该方法类似于一些消息队列如rocketmq和kafka的消息消费方式（consumer定速率的从broker上pull消息），这种方法严格限制了系统的执行速率，起到限流的作用，但对于一些可接受范围内突发的大流量请求也会被限制，导致部分请求延迟过大。

####3.令牌桶（Token Bucket）算法

![令牌桶算法](https://upload-images.jianshu.io/upload_images/3298892-9ccf3924585832d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

初始一个定长的桶，以固定的速率往桶内放令牌，溢出的令牌丢弃，每个请求可以获得任意个数令牌，如果取到了足够令牌就可以继续执行，如果取不到就拒绝该请求。Google的Guava包中的RateLimiter即是以该算法实现，实现比较简单，有兴趣的小伙伴可以自己去研究源码。
该方法算是第一种限制并发数的改进版，取得令牌数是可以动态改变的，释放令牌的数量也不会受限于每个请求的执行时间，而且可以应对可接受范围内的突发流量，应该也是目前最常用的方式。

###三、针对多租户系统的限流方案（仅个人想法）
####1.事先分配
事先为每个租户分配独立的限制量，当然可以简单的平均分配，也可以根据租户事先定制的值分配。这种方法应该是最容易想到，根据总的租户个数平均分配限制量后，再通过对多租户的身份识别分别做流量控制，前面提到的三种算法应该都比较容易通过改进来实现。
缺点：不够灵活，不活跃的租户的量其实是可以暂时分配或者是部分分配给其他租户的。
####2.私有+公有令牌桶
①基于令牌桶算法的改进，每个租户都有一个私有的令牌桶，所有租户有个公有的令牌桶，租户先从公有令牌桶取令牌，取不到再去私有令牌桶取，每个租户有各自独立的放令牌速率，先放到私有桶中，溢出的部分再放到公有桶中，公有桶溢出后就丢弃。这种方式可以保证每个租户都有一个可以得到保障的最低流量，而且还可以将系统资源得到充分的利用。但是这种方案使用时却并不那么美好，因为放令牌操作要遍历每个私有桶，他的时间复杂度是O(n)，相比较原来O(1)的时间复杂度，尤其在海量租户的情况下，严重影响系统效率。
伪代码：

    long timeStamp=getNowTime();
    int publicCapacity;              // 公有桶的容量
    int privateCapacity;            //私有桶的容量
    Map rateMap ;              //每个租户令牌放入速度
    int publicTokens;            //公有桶的当前水量
    Map privateTokensMap;       //私有桶的当前水量

    bool grant(int tenantId, int grantTokens){ //取令牌方法
        //先执行添加令牌的操作
        putTokens();
        //先从公有桶取，再从私有桶取
        int privateTokens = privateTokensMap.get(tenantId);
        if(privateTokens+publicTokens >= grantTokens){
            int tokensDel = publicTokens - grantTokens;
            if(tokensDel >= 0){
                //完全从公有桶取
                publicTokens = tokensDel
            }else{
                //共有桶不够，从私有桶补足
                publicTokens = 0;
                privateTokensMap.put(tenantId,privateTokens+tokensDel);
            }
            return true;
        }else{
            //令牌不够，拒绝请求
            retun false;
        }
    }
    void putTokens(){
        long now = getNowTime();
        long timeDelta = now - timeStamp;
        timeStamp = now;
        for(Map.Entry rateEntry : rateMap){//遍历每个桶，将令牌放入，私有桶溢出的令牌放到公有桶，公有桶溢出的丢弃
            int tenantId = rateEntry.getKey();
            int rate = rateEntry.getValue();
            int privateTokens = privateTokensMap.get(tenantId);
            int tokensDelta = timeDelta*rate;
            if(privateTokens + tokenDelta > privateCapacity){
                rateEntry.setValue(privateCapacity);
                publicTokens = min(publicCapacity, publicTokens+(privateTokens + tokenDelta - privateCapacity));
            }else{
                rateEntry.setValue(privateTokens + tokenDelta);
            }
        }
    }

②在上一种方案的基础下，公有桶也作为一个独立的令牌桶使用，公有桶的令牌流入速率与私有桶不同，每个租户先从公有桶尝试获取，获取不到的情况下，再从私有桶获取，但是每个桶（包括公有桶）的速率总和还是要小于等于系统可接受的最大流量。这样的时间复杂度依然是常数级别的，也可以提高部分闲置资源的使用率。借助现有的限流工具也很容易实现。
伪代码：

    Map<RateLimiter> privateRateLimiterMap;    //租户的私有令牌桶，RateLimiter是普通的令牌桶实现，例如：Guava包里的令牌桶RateLimiter
    RateLimiter publicRateLimiter;   //公有的令牌桶

    void init(Map privateRateMap, int publicRate){ //初始化令牌桶
        publicRateLimiter = RateLimiter.create(publicRate);
        privateRateLimiterMap = new HashMap<>();
        for(Map.Entry rateEntry : privateRateMap){
            RateLimiter privateRateLimiter = RateLimiter.create(rateEntry.getValue());
            privateRateLimiterMap.put(rateEntry.getKey(), privateRateLimiter);
        }
    }

    bool grant(int tenantId, int grantTokens){ //取令牌方法
        if(publicRateLimiter.tryAcquire(grantTokens))
            return true;
        else
            return privateRateLimiterMap.get(tenantId).tryAcquire(grantTokens);
    }