### 第一个事例
logstash首先经过input获取数据，然后经过filter过滤数据，最后output输出数据。    
下面的例子是从文件中获取log，然后经过filter，然后输出到elesticsearch和终端。    
COMBINEDAPACHELOG是预设的过滤器。它的值如下  
<pre><code>
COMBINEDAPACHELOG %{IPORHOST:clientip} %{USER:ident} %{USER:auth} [%{HTTPDATE:timestamp}] "(?:%{WORD:verb} %{NOTSPACE:request} (?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-) %{QS:referrer} %{QS:agent}
</code></pre>
对应的log文件中的log日志为  
<pre><code>
83.149.9.216 - - [04/Jan/2015:05:13:42 +0000] "GET /presentations/logstash-monitorama-2013/images/kibana-search.png HTTP/1.1" 200 203023 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; IntlMac OS X 11_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
</code></pre>
在fitler中有个判断语句，如果日志名称中包含"access"就将type修改为"apache\_access"  
date的作用是将日志的时间替代系统当前时间，这个会影响到建索引，建索引使用的是这个时间。  
geoip是根据IP地址来获取地址信息。grok用于解析非结构化数据。    
<pre><code>
input {
    file {
        path => "/Users/linux/logs/access.log"
        start_position => beginning
    }
}
filter {
    if [path] =~ "access" {
        mutate { replace => { "type" => "apache_access" } }
        grok {
            match => { "message" => "%{COMBINEDAPACHELOG}" }
        }
    }
    date {
        match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
    geoip {
        source => "clientip"
    }
}
output {
    elasticsearch {
        hosts => ["localhost:9200"]
    }
    stdout {  codec => rubydebug  }
}
</code></pre>
上面这个例子在过滤器中只将正常访问的日志给进行了切割，如果是错误日志将不会进行切割，只能看到一个段串。  
下面我们进行修改，对所有日志进行处理。  
<pre></code>
input {
  file {
    path => "/tmp/*_log"
  }
}

filter {
  if [path] =~ "access" {
    mutate { replace => { type => "apache_access" } }
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
    date {
      match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
  } else if [path] =~ "error" {
    mutate { replace => { type => "apache\_error" } }
  } else {
    mutate { replace => { type => "random\_logs" } }
  }
}

output {
  elasticsearch { hosts => ["localhost:9200"] }
  stdout { codec => rubydebug }
}
</code></pre>
我们在这里只是对error和其他的日志类型改了索引的type，没有进行真正的解析。如果要进行解析，还需要对应项目需求来实际操作。  
同样的我们也可以根据条件来进行输出不同的内容。  
下面的例子，我们对不同的访问状态输出了不同的内容。  
<pre><code>
output {
  if [type] == "apache" {
    if [status] =~ /^5\d\d/ {
      nagios { ...  }
    } else if [status] =~ /^4\d\d/ {
      elasticsearch { ... }
    }
    statsd { increment => "apache.%{status}" }
  }
}
</code></pre>


### syslog
处理系统日志，下面的例子输入为tcp、udp的5000端口，然后filter中只处理syslog的数据grok中对数据进行自定义格式化。
<pre><code>
input {
  tcp {
    port => 5000
    type => syslog
  }
  udp {
    port => 5000
    type => syslog
  }
}

filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}

output {
  elasticsearch { hosts => ["localhost:9200"] }
  stdout { codec => rubydebug }
}
</code></pre>
`bin/logstash -f logstash-syslog.conf`  
启动上面的配置后，我们可以`telnet localhost 5000`发送数据进行解析。  
可以发送下面的数据样式。  
`Dec 23 12:11:43 louis postfix/smtpd[31499]: connect from unknown[95.75.93.154]`   





### metadata field
metadata是个隐藏的列，如下是打印不出来metadata的信息的。  
下面的例子是读取终端的输入的信息，然后为信息加上一些我们自己定义的字段  
添加了个一个show字段这个是可以直接输出的，还有2个metadata的字段。这个默认是隐藏的。  
output中使用了if语句，在logstash中可以使用一些简单的语句来进行数据处理，这里判断metadata的test是否为Hello，然后格式化输出。由于我们在filter中把每一个输入都设置成了Hello，所以每一条语句都可以进行输出。  
codec是指定编码格式，这里使用了rubydebug还可以指定json之类的。   
<pre><code>
input { stdin { } }

filter {
  mutate { add_field => { "show" => "This data will be in the output" } }
  mutate { add_field => { "[@metadata][test]" => "Hello" } }
  mutate { add_field => { "[@metadata][no_show]" => "This data will not be in the output" } }
}

output {
  if [@metadata][test] == "Hello" {
    stdout { codec => rubydebug }
  }
}
</code></pre>
如果我们想输出metadata中的内容，只需要修改一下codec的输出设置metadata为true。只有rubydebug才能设置显示metadata.  
`stdout { codec => rubydebug { metadata => true } }`  

在output中可以设置elasticsearch所对应的字段
<pre><code>
output {
  elasticsearch {
    action => "%{[@metadata][action]}"
    document_id => "%{[@metadata][_id]}"
    hosts => ["example.com"]
    index => "index_name"
    protocol => "http"
  }
}
</code></pre>

### 环境变量
logstash配置时可以使用系统环境变量和自带的变量进行配合。  
下面的例子使用了系统环境变量HOME并且我们设置了默认值tmp，如何设置了HOME使用HOME，如果没有使用我们设置的默认值。
<pre><code>
filter {
  mutate {
    add_field => {
      "my_path" => "${HOME:tmp}/file.log"
    }
  }
}

export HOME="/path"
"my_path" => "/path/file.log"

No HOME defined
"my_path" => "/tmp/file.log"
</code></pre>


### 自动重载配置
logstash2.3里增加了自动重载配置，每隔3秒钟去检查一次配置文件是否修改。如果修改过先去检查配置文件是否正确，如果正确直接进行切换。如果错误的话，旧的继续运行，在终端输出新的配置文件的错误。     
<pre><code>
bin/logstash –f apache.config --auto-reload
</code></pre>

### 插件管理
下面的分别为列出所有的插件，列出所有的插件并且包含插件的版本信息，列出所有包含kafka的插件，列出所有output组的插件。  
<pre><code>
bin/logstash-plugin list 
bin/logstash-plugin list --verbose 
bin/logstash-plugin list kafka
bin/logstash-plugin list --group output 
</code></pre>
安装kafka相关的插件
下面分别是直接进行安装，使用打包好的gem文件安装，最后一个是使用源代码编译安装。  
<pre><code>
bin/logstash-plugin install logstash-output-kafka logstash-input-kafka
bin/logstash-plugin install /path/to/logstash-output-kafka-1.0.0.gem
bin/logstash --pluginpath /opt/shared/lib/logstash/input/my-custom-plugin-code.rb
</code></pre>
下面是插件的更新，更新所有插件，更新单个插件，删除单个插件，设置代理。  
<pre><code>
bin/logstash-plugin update 
bin/logstash-plugin update logstash-output-kafka 
bin/logstash-plugin uninstall logstash-output-kafka
export HTTP_PROXY=http://127.0.0.1:3128
</code></pre>


### kafka配置
[kafka input详细配置](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html "kafka")  
[kafka output详细配置](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-kafka.html "kafka")