# 快速接入tp5.0的项目

```
# 安装think-queue
composer require "topthink/think-queue:1.1.*"
# 安装rabbitmq支持
composer require jcleng/think-queue-rabbitmq
```

## 配置

> 配置文件位于 `application/extra/queue.php`,消费类`Job1`使用不变



### 驱动配置

```php
return [
    'connector' => '\jcleng\queue\connector\RabbitMQ',

    'dsn' => '',

    'host' => "127.0.0.1",
    'port' => 5672,

    'vhost' => '/',
    'login' => "guest",
    'password' => 'guest',

    'queue' => "queue_default",

    'options' => [

        'exchange' => [

            'name' => 'queue_exchange',

            /*
            * Determine if exchange should be created if it does not exist.
            */
            'declare' => true,

            /*
            * Read more about possible values at https://www.rabbitmq.com/tutorials/amqp-concepts.html
            */
            'type' => \Interop\Amqp\AmqpTopic::TYPE_DIRECT,
            'passive' => false,
            'durable' => true,
            'auto_delete' => false,
            'arguments' => null,
        ],

        'queue' => [

            /*
            * Determine if queue should be created if it does not exist.
            */
            'declare' => true,

            /*
            * Determine if queue should be binded to the exchange created.
            */
            'bind' => true,

            /*
            * Read more about possible values at https://www.rabbitmq.com/tutorials/amqp-concepts.html
            */
            'passive' => false,
            'durable' => true,
            'exclusive' => false,
            'auto_delete' => false,
            'arguments' => null,
        ],
    ],



];

```

## 创建任务类
> 单模块项目推荐使用 `app\job` 作为任务类的命名空间
> 多模块项目可用使用 `app\module\job` 作为任务类的命名空间
> 也可以放在任意可以自动加载到的地方

任务类不需继承任何类，如果这个类只有一个任务，那么就只需要提供一个`fire`方法就可以了，如果有多个小任务，就写多个方法，下面发布任务的时候会有区别
每个方法会传入两个参数 `think\queue\Job $job`（当前的任务对象） 和 `$data`（发布任务时自定义的数据）

还有个可选的任务失败执行的方法 `failed` 传入的参数为`$data`（发布任务时自定义的数据）

### 下面写两个例子

```
namespace app\job;

use think\queue\Job;

class Job1{

    public function fire(Job $job, $data){

            //....这里执行具体的任务

             if ($job->attempts() > 3) {
                  //通过这个方法可以检查这个任务已经重试了几次了
             }


            //如果任务执行成功后 记得删除任务，不然这个任务会重复执行，直到达到最大重试次数后失败后，执行failed方法
            $job->delete();

            // 也可以重新发布这个任务
            $job->release($delay); //$delay为延迟时间

    }

    public function failed($data){

        // ...任务达到最大重试次数后，失败了
    }

}

```

```

namespace app\lib\job;

use think\queue\Job;

class Job2{

    public function task1(Job $job, $data){


    }

    public function task2(Job $job, $data){


    }

    public function failed($data){


    }

}

```


## 发布任务

> 推送`queue("Job1", ["name"=> "张三"]);`

> `think\Queue::push($job, $data = '', $queue = null)` 和 `think\Queue::later($delay, $job, $data = '', $queue = null)` 两个方法，前者是立即执行，后者是在`$delay`秒后执行

`$job` 是任务名
单模块的，且命名空间是`app\job`的，比如上面的例子一,写`Job1`类名即可
多模块的，且命名空间是`app\module\job`的，写`model/Job1`即可
其他的需要些完整的类名，比如上面的例子二，需要写完整的类名`app\lib\job\Job2`
如果一个任务类里有多个小任务的话，如上面的例子二，需要用@+方法名`app\lib\job\Job2@task1`、`app\lib\job\Job2@task2`

`$data` 是你要传到任务里的参数

`$queue` 队列名，指定这个任务是在哪个队列上执行，同下面监控队列的时候指定的队列名,可不填

## 监听任务并执行

> php think queue:listen

> php think queue:work --daemon（不加--daemon为执行单个任务）

> php think queue:work --queue queue_default --daemon

两种，具体的可选参数可以输入命令加 --help 查看

>可配合supervisor使用，保证进程常驻

- supervisord.conf原文

```conf
[inet_http_server]
port=127.0.0.1:9008

[program:rabbitmq_worker]
command=/Users/jcleng/.nix-profile/bin/php /Volumes/D/runbox/centos7/work/demo/think queue:listen --queue queue_default
directory = /Volumes/D/runbox/centos7/work/demo
user = jcleng
process_name=%(program_name)s_%(process_num)02d
numprocs=4

```
