# PHP实现文本快速查找 - 二分查找法

## 起因
先说说事情的起因，最近在分析数据时经常遇到一种场景，代码需要频繁的读某一张数据库的表，比如根据地区ID获取地区名称、根据网站分类ID获取分类名称、根据关键词ID获取关键词等。虽然以上需求都可以在原始建表时，通过冗余数据来解决。但仍有部分业务存的只是关联表的ID，数据分析时需要频繁的查表。

## 所读的表存在共同的特点
* 数据几乎不会变更
* 数据量适中，从一万到100多万，如果全加载到内存也不太合适。

## 纠结的地方
在做数据分析时，需要十分频繁的读这些表，每秒有可能需要读上万次。其实内部的数据库集群完全可以胜任，但会对线上业务稍有影响。（你懂得，小公司不可能为离线分析做一套完整的数据存储服务。大部分数据分析还要借助线上的数据集群）

## 优化方案的思考
有没有一种方式可以不增加线上的压力，同时提供更高效的查询方式？想过redis，但最终选择用文本存储。因为数据分析是一个独立的需求，不希望与现有的redis集群或者其它存储服务有交集。还有一个原因是每次分析的中间结果，对下一次分析并没有很大的实质作用，并不需要把结果持久存储，而且占的内存也会较多。最终使用文本存储，然后用二分来查找。特点，1，存储非常快，虽然redis等nosql服务虽然已经非常快，但仍无法与文本存储相提并论；2，查找的时候使用二分查找，百万条记录查询也可在0.1ms内完成（使用线上的普通硬盘，如果是ssd盘会更快）。

## 实现步骤
* 将数据库中需要的字段导出到文本
	
		方法：使用mysql的phpmyadmin工具，执行sql语句查出主建id和相应字段
		如以上的关键词表： select kid, keyword from keyword
		然后使用phpmyadmin的导出工具，可以快速把结果导出到文本中
		操作截图：
![image](http://ocggi1ecj.bkt.clouddn.com/0F48BF41-7D01-48F9-8064-52EF5871FFBA.png)

------------------
		 
![image](http://ocggi1ecj.bkt.clouddn.com/79421214-E93A-46D3-9148-5D9F2A68F25E.png)

* 将导出的文本（已经按id进行过排序）转换格式重新存储
* 程序读取转换后的格式

## 文本存储格式
说明 ：需求中，文本每行有两列，第一列是主建ID（数字），第二列为文本。整个文本已经按第一列有序排列，两列之间用tab键分隔。
之前有看过ip.dat的存储，本次仿照其存储格式：将文本中的内容每行转换为固定长度后，存储到新的文件。搜索时，使用文件操作函数fopen，fseek，fgets等函数按字节读取内容，并以二分查找法快速定位需要的内容。

## 代码实现部分
* 通用类，类似需求只需要提供符合标准的文本（每行两列，第一列为查找的ID，第二列为文本。同时文本已经按第一列有序排序）
* 生成以上所提到的存储格式
* 提供根据id查询接口

## 代码片断
* 重新生成新的存储格式

		//读源文件，写入到新的索引文件
        $readfd = fopen($this->filename, 'rb');
        $writefd = fopen($this->formatFile.'_tmp', 'wb+');
        if ($readfd === false || $writefd === false) {
            return false;
        }
        echo "\n start reformat file $this->filename ..";
        while (!feof($readfd)) {
            $line = fgets($readfd, 8192);
            fwrite($writefd, pack("a".$this->maxLength, $line));
        }
        echo "\n reformat ok\n";
        fclose($readfd);
        fclose($writefd);
        rename($this->formatFile.'_tmp', $this->formatFile);
* 二分查找的代码片断
	    
	    /**
        * 在索引文件中进行二分查找
        * @param  int $id    进行二分查找的id
        * @return [type]     [description]
        */
        public function search($key)
        {
            $filesize = filesize($this->formatFile);
            $fd = fopen($this->formatFile, "rb");
            $left = 0; //行号
            $right = ($filesize / $this->maxLength) - 1; 
            while ($left <= $right) {
                $middle = intval(($right + $left)/2);
                fseek($fd, ($middle) * $this->maxLength);
                $info = unpack("a*", fread($fd, $this->maxLength))['1'];
                $lineinfo = explode("\t", $info, 2);
                if ($lineinfo['0'] > $key) {
                    $right = $middle - 1;
                } elseif ($lineinfo['0'] < $key) {
                    $left = $middle + 1;
                } else {
                    return $lineinfo['1'];
                }
            }
            return false;
        }
* 整个类库代码一共91行，具体可查看github的demo代码

## 运行截图 
![image](http://ocggi1ecj.bkt.clouddn.com/ss.png)
以上拿100万的关键词进行测试，根据关键词id快速查找关键词，平均速度可以达到0.1毫秒。

