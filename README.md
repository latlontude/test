#iris常用运维处理
[TOC]
##   1. 增加Topic
*   进入*iris_manager* 所在的部署目录 */home/work/services/iris-manager/client/conf*
*   找到配置文件 *route_config.xml*
*   其配置文件描述如下:
  ```xml
  <?xml version="1.0" encoding="utf-8" ?>
<route version="1234">
    <!-- all route定义 -->
    <topics>
        <topic id="1093" sub_hash_cnt="2" writer_log_prefix="new_miliao_follow_">
            <shard begin="0" end="1000" ip="172.28.0.9" port="9402" status="0" />
        </topic>
        <topic id="1090" sub_hash_cnt="2" writer_log_prefix="new_miliao_chat_msg">
            <shard begin="0" end="1000" ip="172.28.0.9" port="9402" status="0" />
        </topic>
        <topic id="1088" sub_hash_cnt="9" writer_log_prefix="miiao_v2_group_msg_c2s">
            <shard begin="0" end="1000" ip="172.28.0.9" port="9402" status="0" />
        </topic>
        <topic id="1091" sub_hash_cnt="9" writer_log_prefix="miliao_v2_group_push">
            <shard begin="0" end="1000" ip="172.28.0.9" port="9402" status="0" />
        </topic>
    </topics>
    <hosts>
        <host ip="172.28.0.9" port="9402" status="0" />
    </hosts>
    <time v="2015/06/15 14:26:39" />
</route>
  ```

*   仿照上面代码中的示例，增加一个*topic*，注意*sub_hash_cnt*，表示流水子队列的数目（**每个子队列只对应一个推送进程，如果流水特别多的，如果子队列设置比较少，则可能导致推送阻塞，因为推送进程太少**），取值范围为**2~9**，对于产生流水较多的，适合配置多一些，比如9，对于流水较少的，可以配置少一些，比如2。 ip表示流水写入的机器*IP*，即*iris_writer的机器*，目前就一台机器。
*   增加新*topic*后，注意要**修改route version（增加其值）**，*version*版本号增加，这样*iris_acc*跟*iris_manager*通信时，发现路由版本号增加，就会拉取新的路由。
*   重启*iris_manager*，实质上*iris_manager*会定期load该文件，可以不用重启，但为了保险起见，可以重启*iris_manager*。
*   进入*iris_writer*的配置文件目录*/home/work/services/iris-writer/client/conf*,查看*topic.conf*，如果其版本号跟*iris_manager*的一致，则说明路由信息已经同步到*iris_writer*。
*   测试业务逻辑模块写入新增的topic数据，查看iris_acc的统计（每隔1分钟打印一次），看看新的topic数据的流水是否成功。

##   2. 增加消费者
*   进入*iris_manager* 所在的部署目录 */home/work/services/iris-manager/client/conf*
*   找到配置文件*consumer_config.json*
*   其配置文件格式如下（下面是部分摘要信息，完整信息可以查看文件的具体类容）：
```json
{
    "version": 527,
    "sopath": "../bin/so/live_so.so",
    "consumers": [
     {

            "consumer_id": 50284,
            "subversion": 1,
            "thread_cnt": "1",
            "timeout": "1000",
            "ip": "0",
            "remote_sub_mod_id": "801310001",
            "l5id": "72001",
            "remote_mod_id": "801310000",
            "cmd_id": "6009",
            "pre": "new_miliao_chat_msg",
            "sub_hash_cnt": "2",
            "topic_id": "1090",
            "sleep_step": "0",
            "batch_cnt": "1",
            "retry_cnt": "-1",
            "port": "0",
            "l5_cmd_id": "1001"
        },
        {
            "consumer_id": 50286,
            "subversion": 3,
            "thread_cnt": "1",
            "timeout": "1000",
            "ip": "0",
            "remote_sub_mod_id": "801330001",
            "l5id": "72002",
            "remote_mod_id": "801330000",
            "cmd_id": "6009",
            "pre": "new_miliao_chat_msg",
            "sub_hash_cnt": "2",
            "topic_id": "1090",
            "sleep_step": "0",
            "batch_cnt": "1",
            "retry_cnt": "3",
            "port": "0",
            "l5_cmd_id": "1001"
        }
    ]
}
```
*   仿照上面代码中的示例，**增加一个consumer_id，不能重复，填写该消费者的topic_id和对应的sub_hash_cnt/pre信息**。注意**sub_hahs_cnt数目确认之后就不能变化**。*remote_sub_mod_id/remote_mod_id*是模块间调用的信息，目前系统里面没有用到，可以随便填**l5id要填写消费者对于的L5ID**。
*   ***修改version，增加其值***，原理类似新增topic的操作。
*   ***再次检查下新增的信息正确***，比如*consumer*对应的*topic*的*sub_hash_cnt/pre*，消费者对于的*L5ID*，失败重试的次数等。
*   **重启iris_manager**
*   进入*iris_notify_server*的配置文件目录*/home/work/services/iris-notify-server/iris-notify-server-online-v3/*，查看其*s_server.conf*中配置文件中的版本号是否跟刚才的修改后的版本一致。
*   注意*s_server.conf*中的*consumer_id*的格式，*consumer_id*是有加后缀，后缀是*sub_hash_cnt*的编号，比如502841/502842是在50284后加上了编号1和2。
```xml
<?xml version="1.0" encoding="utf-8" ?>
<root>
       <work_self>
                <consume_id value="502841" />
                <subversion value="1" />
                <thread_cnt value="1" />
                <timeout value="1000" />
                <ip value="0" />
                <remote_sub_mod_id value="801310001" />
                <l5id value="72001" />
                <remote_mod_id value="801310000" />
                <cmd_id value="6009" />
                <topic_id value="1090" />
                <sleep_step value="0" />
                <batch_cnt value="1" />
                <retry_cnt value="-1" />
                <port value="0" />
                <l5_cmd_id value="1001" />
       </work_self>
        
        <work_self>
                <consume_id value="502842" />
                <subversion value="1" />
                <thread_cnt value="1" />
                <timeout value="1000" />
                <ip value="0" />
                <remote_sub_mod_id value="801310001" />
                <l5id value="72001" />
                <remote_mod_id value="801310000" />
                <cmd_id value="6009" />
                <topic_id value="1090" />
                <sleep_step value="0" />
                <batch_cnt value="1" />
                <retry_cnt value="-1" />
                <port value="0" />
                <l5_cmd_id value="1001" />
        </work_self>
        
        
        <work_self>
                <consume_id value="502861" />
                <subversion value="3" />
                <thread_cnt value="1" />
                <timeout value="1000" />
                <ip value="0" />
                <remote_sub_mod_id value="801330001" />
                <l5id value="72002" />
                <remote_mod_id value="801330000" />
                <cmd_id value="6009" />
                <topic_id value="1090" />
                <sleep_step value="0" />
                <batch_cnt value="1" />
                <retry_cnt value="3" />
                <port value="0" />
                <l5_cmd_id value="1001" />
         </work_self>
</root>        
```


*   **重启notify_server**,重启之后就会向新的消费者推送流水。本质上不用重启，但为了保险起见，可以重启下。
*   **ps -ef |grep CHILD |grep 50292** 查看推送进程是否存在，这里50292替换为你新增的消费者ID即可，注意的进程名称如下:
```shell
[root@bsy-fk4 bin]# ps -ef |grep CHILD |grep 50292
root     30165 30160  0  2018 ?        20:04:40 502921_CHILD_notify_server
root     30166 30160  0  2018 ?        19:49:21 502922_CHILD_notify_server
root     30167 30160  0  2018 ?        20:19:05 502923_CHILD_notify_server
root     30168 30160  0  2018 ?        20:00:03 502924_CHILD_notify_server
root     30169 30160  0  2018 ?        19:54:03 502925_CHILD_notify_server
root     30170 30160  0  2018 ?        19:40:59 502926_CHILD_notify_server
root     30171 30160  0  2018 ?        19:48:00 502927_CHILD_notify_server
root     30172 30160  0  2018 ?        19:36:17 502928_CHILD_notify_server
root     30173 30160  0  2018 ?        19:48:42 502929_CHILD_notify_server
```

   其中的进程名称是50292[1-9], 这里是因为该*topic*有9个子队列,每个子队列对应一个推送进程，所以有9个进程对应该消费者。如果对应的*topic*有2个子队列，则推送进程就是2个，比如下面的50286这个消费者：
```shell
[root@bsy-fk4 bin]# ps -ef |grep CHILD |grep 50286
root     30163 30160  0  2018 ?        19:10:39 502861_CHILD_notify_server
root     30164 30160  0  2018 ?        18:40:30 502862_CHILD_notify_server
```

## 3. 重新给某个消费者推送流水-方法1
> 这种场景一般是业务逻辑出现有BUG，导致之前的消费流水处理不正确，需要重新消费一次流水。

*   进入iris_server的部署目录下pos文件夹，*/home/work/services/iris-notify-server/iris-notify-server-online-v3/pos*，里面记录了各消费者推送的位置信息。
*   比如消费者ID50295,其推送进程有9个，每个推送进程一个pos文件，如下图所示：
```shell
[root@bsy-fk4 pos]# ls -l work50295*
-rw------- 1 root root 24 4月  25 10:04 work502951.pos
-rw------- 1 root root 24 4月  25 10:04 work502952.pos
-rw------- 1 root root 24 4月  25 10:03 work502953.pos
-rw------- 1 root root 24 4月  25 10:04 work502954.pos
-rw------- 1 root root 24 4月  25 10:03 work502955.pos
-rw------- 1 root root 24 4月  25 10:04 work502956.pos
-rw------- 1 root root 24 4月  25 10:04 work502957.pos
-rw------- 1 root root 24 4月  25 10:04 work502958.pos
-rw------- 1 root root 24 4月  25 10:03 work502959.pos
```
*    修改这些推送进程的pos位置信息，然后重启*notify_server*。
*    **如何修改pos文件的方法**? 进入fk4下的*/data/iris_tools*，利用工具*iris_pos*来进行修改，*iris_pos*可以查看pos文件信息，也可以修改pos文件信息。
```shell
[root@bsy-fk4 iris_tools]# ./iris-pos
	usage
		./iris-pos filename # display pos info
		./iris-pos filename 2018-09-04 10:10 # set pos info
[root@bsy-fk4 iris_tools]# ./iris-pos  work502951.pos
  current: 2019-04-25 10:00:00 @7524
  base:    2019-04-25 10:00:00 @7524
```
*   **注意pos文件是按时间来标识的，最小单位是小时**，比如可以把pos设置到2018-09-04 10:00。
*   
##   4. 重新给某个消费者推送流水-方法2(推荐使用)
*   进入fk4下的*/data/iris_tools*，利用工具iris来进行推送。
*   iris工具的使用方法:
```shell
[root@bsy-fk4 iris_tools]# ./iris
usage
	./iris filename host port topic
```
*   指定流水文件路径及*topic_id*，推送的目标IP/PORT。目前iris的流水都是一个小时一个文件，可以把需要重放的流水文件拷贝到过来，然后一一推送。

## 5. 单聊和群聊的消息流水分析工具
*   单聊消息流水分析工具*iris-chat*
```shell
[root@bsy-fk4 iris_tools]# ./iris-chat //home/work/MISTORE/writer_bill_9402/1090/1/new_miliao_chat_msg_20190425_10.bill |head -n 100
from: 1001409
to: 1000038
seq: 1000009
timestamp: 1556157600247
cid: 1556157606636165
msg_type: 1
msg_body: "Gary"
msg_ext: ""
msg_status: 0
from_appid: 10004
13: 1027

from: 1000329
to: 1000037
seq: 1000856
timestamp: 1556157603886
cid: 1556157602848302
msg_type: 1
msg_body: "新来的猫🐱很可爱，很能吃，很活泼"
msg_ext: ""
msg_status: 0
from_appid: 10004
13: 1006

from: 1000037
to: 1000329
seq: 1000857
timestamp: 1556157613749
cid: 1556157613927377
msg_type: 1
msg_body: "昨天晚上带去 医院检查了吗"
msg_ext: ""
msg_status: 0
from_appid: 10004
13: 1085
```
*   群聊消息流水分析工具*iris-group*，类似单聊流水分析工具。
