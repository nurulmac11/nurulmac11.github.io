+++
title = "Curse of AMQP: Celery"
date = 2023-10-30
description = "How to solve annoying problems with celery."

[taxonomies]
tags = ["celery", "eta", "sqs", "eta", "retry"]
+++
> Nobody panics when things go "according to plan." Even if the plan is horrifying! 

If you haven't had a problem with celery yet, most likely, you haven't used it enough. In this post, I will explain problems that I've encountered and solutions I finally found after struggling for days.

<!-- more -->
 TLDR:
 - For a low workload of a few processes per minute, stick with default thread pooling for your workers.
 - If you're dealing with high throughput (thousands of processes per minute), opt for eventlet pooling.
 - Avoid using gevent.
 - When facing memory issues, ensure there are no unhandled exceptions within your tasks and task methods return only True, also, don't use exc parameter within retry calls.

---

This was our celery config and command to start workers within fastapi project;

```python
app = Celery(
    "tasks",
    broker=os.getenv("CELERY_BROKER", "redis://redis"),
    broker_connection_retry=True,
    broker_connection_max_retries=None,
    broker_connection_timeout=60,
    result_backend=os.getenv("CELERY_RESULT_BACKEND", "db+postgresql://admin:admin@192.168.1.1:5432/example-db"),
    task_soft_time_limit=3,
)

app.conf.broker_transport_options = {
    "visibility_timeout": 12 * 3600,
}
```

```shell
celery -A src.tasks worker -n hp@Billing-Manager --pool=gevent --concurrency=100 -Q hp -l INFO &
```


Pretty simple and straightforward actually. We have redis as a broker on the same network and postgresql db for storing results of tasks. Also we are using ETA and retry features of celery which makes things more complex and hard. But we keep having mysterious problems with celery. What's happening was, our workers were stopping to process tasks suddenly, after working several hours, without any error. When we check the logs, status of the server, network, everything seemed ok. I even tried switching in between redis and rabbitmq as broker several times, but didn't help at all.

There were <a href="https://stackoverflow.com/a/33936673/4087794" target="_blank">some debugging suggestions</a> on the internet using tools such as strace but they lead to nothing. My best guess was it was caused some kind of deadlock or connection problem with broker. Seemed like celery was losing connection to broker but was unable to reconnect to broker. I found some discussions on celery/kombu repo's discussion pages but there were no solution, just some unclosed pull requests that still needs work.

---

```shell
celery -A src.tasks worker -n hp@Billing-Manager --concurrency=20 -Q hp -l INFO &
```

Then I started to try different methods. Firstly, I tried to switch just thread pooling. We were using gevent, which is using green threads, which allows you to use hundreds of thread compared to the default thread pooling but as expected it was working very slow and we were having oom(out of memory) issues.


```shell
celery -A src.tasks worker -n 'hp@Billing-Manager' --pool=eventlet --concurrency=100 -Q hp -l INFO &
```

Then I found another pooling method, which was eventlet. It's very similar to gevent, I am still not sure what's the difference exactly but it seems like they are using different libraries underneath them. If you want to read more about this, you can <a href="https://blog.gevent.org/2010/02/27/why-gevent/" target="_blank">check this link</a>.

This was working much better compared to the alternatives but it was filling memory just within minutes. After checking task methods, I cleared all unhandled exceptions and return values. Also removed exc parameter from self.retry calls. Even we had a database for storing results, it seems like celery was unable to clear these from memory or it was happening because our tasks were retried all day several times. But this memory issue also solved after this update. Now, it works flawlessly.

That was all. Hope that works for you too. Reach me out via linkedin if you have any questions.
