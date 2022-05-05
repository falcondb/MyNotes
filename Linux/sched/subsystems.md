## UClamping

### [Uclamping V5 patch](https://lore.kernel.org/lkml/20181029183311.29175-1-patrick.bellasi@arm.com/)

### [A blog](https://www.1024sou.com/article/589657.html)



## Load balancing

### [CFS任务的负载均衡（框架篇）](https://blog.csdn.net/feelabclihu/article/details/105502168?spm=1001.2014.3001.5502)
 - three scenarios could trigger load balancing
  - load balance (task migration) at 1) tick; 2) pick next; 3) waking up idle CPUs through IPI
  - task placement
  - active upmigration / misfit task
