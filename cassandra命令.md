
<!-- http://cassandra.apache.org/doc/cql3/CQL.html#createTableStmt -->

Cassandra在CQL语言层面支持多种数据类型。
CQL类型	对应Java类型	描述
ascii	String	ascii字符串
bigint	long	64位整数
blob	ByteBuffer/byte[]	二进制数组
boolean	boolean	布尔
counter	long	计数器，支持原子性的增减，不支持直接赋值
decimal	BigDecimal	高精度小数
double	double	64位浮点数
float	float	32位浮点数
inet	InetAddress	ipv4或ipv6协议的ip地址
int	int	32位整数
list	List	有序的列表
map	Map	键值对
set	Set	集合
text	String	utf-8编码的字符串
timestamp	Date	日期
uuid	UUID	UUID类型
timeuuid	UUID	时间相关的UUID
varchar	string	text的别名
varint	BigInteger	高精度整型

<!-- 启动cassandra客户端工具 -->
./cassandra/cqlsh 127.0.0.1 9042 

// 显示keyspace，keyspace相当于MySQL中的database。
desc keyspaces;


<!-- 如果没有该keyspace，则创建，可以设置有策略和有多少份拷贝 -->
CREATE KEYSPACE IF NOT EXISTS finedu_message WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '2'}  AND durable_writes = true;

<!-- 选择刚才创建的keyspace -->
use findeu_message

<!-- 创建一个表 -->
CREATE TABLE sms_template (
  id text,
  msg text,
  title text,
  PRIMARY KEY (id) );

<!--  插入一条数据 -->
INSERT INTO sms_template (id,msg,title) VALUES ('1','error', 'error');
select * from sms_template;

<!-- 条件查询 -->
select * from sms_template where id='1';

select * from sms_template where msg='1';
InvalidRequest: code=2200 [Invalid query] message='No supported secondary index found for the non primary key columns restrictions'
可以看到会报错，cassandra的where语句中必须是有索引的。

<!-- 创建索引 -->
create index on sms_template(msg);
select * from sms_template where msg='1';


<!-- 更新数据 -->
update sms_template set title = 'err' where msg = 'error';
delete from sms_template where msg='1';
InvalidRequest: code=2200 [Invalid query] message='Some partition key parts are missing: id'
可以看到更新和删除使用非主键的条件都会报错，where后面必须跟主键。
Cassandra 不支持双引号。


<!-- 创建一个新的类型，然后使用自己定义的类型创建表 -->

CREATE type weixin_message_article (
    title text,
    description text,
    imageUrl text,
    url text );


CREATE TABLE weixin_message (
  eventType text,
  keyword text,
  msgType int,
  enabled boolean,
  content text,
  <!-- items list<frozen<articles>>, -->
  PRIMARY KEY (eventType, keyword) );

create index on weixin_message(enabled);

INSERT INTO weixin_message (keyword,msgType,enabled,content,items) VALUES ('test',1, true,'test', [{title: 'test', description: 'test', imageUrl: 'test', url: 'test'},{title:'test', description:'test', imageUrl:'test', url:'test'}]);

INSERT INTO weixin_message (keyword,msgType,enabled,content,items) VALUES ('test',1, true,'test', [('test', 'test', 'test', 'test'), ('test', 'test', 'test', 'test')]);

INSERT INTO weixin_message (keyword,msgType,enabled,content) VALUES ('test',1, true,'test');


UPDATE weixin_message SET items = [(1, 'test', 'test', 'test', 'test'), (1, 'test', 'test', 'test', 'test')] WHERE keyword='test';


