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
<pre><code>
input {
    file {
        path => "/Users/linux/logs/tmp.log"
        start_position => beginning
    }
}
filter {
    grok {
        match => { "message" => "%{COMBINEDAPACHELOG}"}
    }
    geoip {
        source => "clientip"
    }
}
output {
    elasticsearch {}
    stdout {}
}
</code></pre>

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

