### **syncClient**

>   syncClient，数据实时同步中间件（同步mysql到kafka、redis、elasticsearch、httpmq）！

 本项目使用canal，将mysql的表数据实时同步到kafka、redis、elasticsearch、httpmq；  
 
 基本原理：    
 canal解析binlog的数据，由syncClient订阅，然后实时推送到kafka或者redis、elasticsearch、httpmq；如果kafka、redis、es、httpmq服务异常，syncClient会回滚操作；canal、kafka、redis、es、httpmq的异常退出，都不会影响数据的传输；


---

**目录：**  
bin：已编译二进制项目，可以直接使用；  
src：源代码；  

---

**配置说明：**

#common  
system.debug=1          # 是否开始调试：1未开启，0为关闭（线上运行请关闭）  

#canal server  
canal.ip=127.0.0.1      # canal 服务端 ip;  
canal.port=11111        # canal 服务端 端口：默认11111;  
canal.destination=one   # canal 服务端项目（destinations），多个用逗号分隔，如：redis,kafka;    
canal.username=         # canal 用户名：默认为空;     
canal.password=         # canal 密码：默认为空;  

#redis plugin  
redis.target_type=redis  # 同步插件类型 kafka or redis、elasticsearch、httpmq;   
redis.target_ip=         # redis服务端 ip;   
redis.target_port=       # redis端口：默认6379;   
redis.target_deep=       # 同步到redis的队列名称规则;  
redis.target_filter_api= # rest api地址，配置后会根据api返回的数据过滤同步数据;   

#kafka plugin  
kafka.target_type=kafka  # 同步插件类型 kafka;    
kafka.target_ip=         # kafka服务端 ip;   
kafka.target_port=       # kafka端口：默认9092;   
kafka.target_deep=       # 同步到kafka的集合名称规则;  
kafka.target_filter_api= # rest api地址，配置后会根据api返回的数据过滤同步数据;    

#elasticsearch plugin  
es.target_type=elasticsearch  # 同步插件类型elasticsearch;    
es.target_ip=10.5.3.66        # es服务端 ip; 
es.target_port=               # es端口：默认9200; 
es.target_deep=               # 同步到es的index名称规则;  
es.target_filter_api= # rest api地址，配置后会根据api返回的数据过滤同步数据;   

#httpmq plugin  
httpmq.target_type=httpmq    # 同步插件类型 httpmq;    
httpmq.target_ip=10.5.3.66   # httpmq服务端 ip; 
httpmq.target_port=1218      # httpmq端口：默认 1218  
httpmq.target_deep=          # 同步到httpmq的队列名称规则;  
httpmq.target_filter_api= # rest api地址，配置后会根据api返回的数据过滤同步数据;   

#cache plugin  
cache.target_type=cache         # 缓存同步插件
cache.target_plugin=memcached   # 缓存同步类型：暂支持redis、memcached缓存服务器;  
cache.target_ip=127.0.0.1       # 缓存服务器ip;   
cache.target_port=11211         # 缓存服务器端口;   
cache.target_filter_api=        # rest api地址，配置后会根据api返回的数据过滤同步数据;  
cache.target_version_sign=      # 缓存key前缀  

#target_deep参数影响topic规则，默认值1： 
1、sync_{项目名}_{db}_{table}; 
2、sync_{项目名}_{db};
3、sync_{项目名}; 
4、sync_{db}_{table}; 

---

**使用场景(基于日志增量订阅&消费支持的业务)：**

数据库镜像  
数据库实时备份  
多级索引 (分库索引)  
search build  
业务cache刷新  
数据变化等重要业务消息  

**Kafka：**

Topic规则：对应配置项目target_deep指定的规则，比如：target_deep=4，数据库的每个表有单独的topic，如数据库admin的user表，对应的kafka主题名为：sync_admin_user  
Topic数据字段：  

	插入数据同步格式：
    {
        "head": {
            "binlog_pos": 53036,
            "type": "INSERT",
            "binlog_file": "mysql-bin.000173",
            "db": "sdsw",
            "table": "sys_log"
        },
        "after": [
            {
                "log_id": "1",
            },
            {
                "log_ip": "27.17.47.100",
            },
            {
                "log_addtime": "1494204717",
            }
        ]
    }
	
	修改数据同步格式：
    {
        "head": {
            "binlog_pos": 53036,
            "type": "UPDATE",
            "binlog_file": "mysql-bin.000173",
            "db": "sdsw",
            "table": "sys_log"
        },
        "before": [
            {
                "log_id": "1",
            },
            {
                "log_ip": "27.17.47.100",
            },
            {
                "log_addtime": "1494204717",
            }
        ],
        "after": [
            {
                "log_id": "1",
            },
            {
                "log_ip": "27.17.47.1",
            },
            {
                "log_addtime": "1494204717",
            }
        ]
    }
	
	删除数据同步格式：
    {
        "head": {
            "binlog_pos": 53036,
            "type": "DELETE",
            "binlog_file": "mysql-bin.000173",
            "db": "sdsw",
            "table": "sys_log"
        },
        "before": [
            {
                "log_id": "1",
            },
            {
                "log_ip": "27.17.47.1",
            },
            {
                "log_addtime": "1494204717",
            }
        ]
    }

head.type 类型：INSERT（插入）、UPDATE（修改）、DELETE（删除）； 

head.db 数据库； 

head.table 数据库表；

head.binlog_pos  日志位置； 

head.binlog_file 日志文件；  

before： UPDATE（修改前）、DELETE（删除前）的数据；  

after：  INSERT（插入后）、UPDATE（修改后）的数据；  


**Redis：**

List规则：对应配置项目target_deep指定的规则，比如：target_deep=4，数据库的每个表有单独的list，如数据库admin的user表，对应的redis list名为：sync_admin_user  

**Elasticsearch**

规则：数据库的每个表有单独的Elasticsearch index，如数据库admin的user表，对应的es index名为：sync_admin_user, index type 为default;

Elasticsearch同步数据的head中有id字段；  

Mysql 同步到 Elasticsearch注意事项：

1、表需要有一个唯一id主键；  
2、表时间字段datetime会转为es的时间字段，其他字段对应es的文本类型；  
3、主键、时间字段禁止修改，其他字段尽量提前规划好；   

**Httpmq：**

List规则：对应配置项目target_deep指定的规则，比如：target_deep=4，数据库的每个表有单独的list，如数据库admin的user表，对应的redis list名为：sync_admin_user  


**Cache：**

缓存同步插件：原理是根据数据库变更同步更新表及字段的版本号，业务sdk根据版本号变化判断是否需要更新数据。同步开发了缓存配置管理中心、缓存版本调用sdk（未开源）；  


<a href="https://info.flagcounter.com/AEYx"><img src="https://s11.flagcounter.com/count2/AEYx/bg_FFFFFF/txt_000000/border_CCCCCC/columns_2/maxflags_10/viewers_0/labels_1/pageviews_1/flags_0/percent_0/" alt="Flag Counter" border="0"></a>

