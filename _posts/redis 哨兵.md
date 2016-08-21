---
layout: default
title: redis哨兵
description: redis哨兵是其高可用性的解决方案，介绍哨兵。
categories: [redis]
tags: [redis, 哨兵，sentinel]
---

# redis sentinel 哨兵（未完成）

源码来自redis2.8.24

redis哨兵是其高可用性的解决方案：由一个或多个哨兵可以监视任意多个主服务器，以及这些主服务器的从机，在被监视的主机下线时，自动将其的可用从机代替主机。

哨兵其实就是一台运行在特殊模式下的redis服务器，它的启动方式和服务器略有不同。

### 哨兵的初始化

```c
//redis.c 3282行
int main(int argc, char **argv) {
	 //略...
	 //redis.c 3295行
    server.sentinel_mode = checkForSentinelMode(argc,argv); //判断程序名是否为：redis-sentinel
    initServerConfig();

    /* We need to init sentinel right now as parsing the configuration file
     * in sentinel mode will have the effect of populating the sentinel
     * data structures with master nodes to monitor. */
    if (server.sentinel_mode) { //运行在哨兵模式下，读取配置，启动哨兵
        initSentinelConfig(); //仅仅只是改变port
        initSentinel(); //读取哨兵的配置
    }
    //略...
```

在initSentinel()方法中会抹去redis服务器的命令转而注册哨兵该支持的命令。

### 哨兵支持的命令

```c
//sentinel.c 390行
struct redisCommand sentinelcmds[] = {
    {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
    {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
    {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},
    {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
    {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},
    {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
    {"publish",sentinelPublishCommand,3,"",0,NULL,0,0,0,0,0},
    {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0},
    {"role",sentinelRoleCommand,1,"l",0,NULL,0,0,0,0,0},
    {"shutdown",shutdownCommand,-1,"",0,NULL,0,0,0,0,0}
};

//sentinel.c 410行
/* Perform the Sentinel mode initialization. */
void initSentinel(void) {
    unsigned int j;

    /* Remove usual Redis commands from the command table, then just add
     * the SENTINEL command. */
    dictEmpty(server.commands,NULL); //抹去redis服务器的命令
    for (j = 0; j < sizeof(sentinelcmds)/sizeof(sentinelcmds[0]); j++) {
        int retval;
        struct redisCommand *cmd = sentinelcmds+j;

        retval = dictAdd(server.commands, sdsnew(cmd->name), cmd); //注册哨兵该支持的命令
        redisAssert(retval == DICT_OK);
    }
	 //略...
```