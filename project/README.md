## 一、引言

### 简介

​    当代常常听到说，现在的代码，与古代的唐诗和宋词一样重要。对于我们这些代码人来说，多写几行代码没什么，但真要写出一首诗，问题就大了。

​    那为什么我们不去尝试一下用代码写出一首自己想要的诗来？

  ### 开发环境

+ SQL Server 2019
+ python 3.8+flask
+ Ubuntu 20.04
+ 微信开发者工具

### 数据库

​    本次数据来源为https://github.com/Werneror/Poetry

​    选取了其中的汉(363)，魏晋(3020)，南北(4586)，隋(1170)，唐(49195)，宋(287114)，元(37375)共计382823条数据



## 二、数据预处理

​    下面我们以五言诗词为例，进行预处理。七言诗词的操作方式相同。
   

### 合并所有诗歌并去除异常值

```sql
select * into shige_full
from (select * from song 
where contents not like '%□%' 
and ttitle not like '%□%'
and contents not like '%?%'
and ttitle not like '%?%'
union
select * from tang
where contents not like '%□%' 
and ttitle not like '%□%'
and contents not like '%?%'
and ttitle not like '%?%'
union
select * from han
where contents not like '%□%' 
and ttitle not like '%□%'
and contents not like '%?%'
and ttitle not like '%?%'
union
select * from nanbei
where contents not like '%□%' 
and ttitle not like '%□%'
and contents not like '%?%'
and ttitle not like '%?%'
union
select * from sui
where contents not like '%□%' 
and ttitle not like '%□%'
and contents not like '%?%'
and ttitle not like '%?%'
union
select * from weijin
where contents not like '%□%' 
and ttitle not like '%□%'
and contents not like '%?%'
and ttitle not like '%?%'
union
select * from yuan
where contents not like '%□%' 
and ttitle not like '%□%'
and contents not like '%?%'
and ttitle not like '%?%') a
```

### 拆分出五言与七言诗歌

```sql
create view wuyan as
select * from shige_full
where contents like '[吖-座][吖-座][吖-座][吖-座][吖-座][，。]%'

create view qiyan as
select * from shige_full
where contents like '[吖-座][吖-座][吖-座][吖-座][吖-座][吖-座][吖-座][，。]%'
```

### 分词

​    由于sql语言并不能精准的做到分词，只能通过词频统计的方式进行“大致分词”，所以在此我们采用python的thulac库，对古诗进行分词，参考网址：http://thulac.thunlp.org/demo

​    示例：随机过程随机过

​    结果：随机\_v 过程\_n 随机\_v 过\_u

```python
import thulac
import pandas as pd

df = pd.read_csv('wuyan.csv', encoding = 'gb18030')
thul = thulac.thulac()

for i in range(len(df)):
    text = df['contents'][i]
    text_res = thul.cut(text, text = True)
    df.loc[i,'fenci'] = text_res
print(df['fenci'][0])
df.to_csv('wuyan_fenci.csv', encoding = 'gb18030')
```

### 词频统计

​    直接利用sql语句完成词频统计即可

```sql
declare @str nvarchar(200)

if object_id('tempdb.dbo.#array') is not null  --判断临时表是否已经存在
   drop table #array
create table #array(ch nvarchar(20))

--全部改成qiyan即可创建七言的词频库
declare cur scroll cursor for 
    select fenci from wuyan_fenci
open cur
fetch first from cur into @str
while @@FETCH_STATUS = 0
begin
    insert into #array(ch) select value from
    string_split(@str,' ')
    where patindex('%[吖-座]%', value)>0
	fetch next from cur into @str
end
close cur
deallocate cur

select ch, count(*) cnt into wuyan_cipin 
from #array group by ch order by cnt desc
drop table #array

select * from wuyan_cipin order by cnt desc
```

### 韵脚

​    我们以每个字的最后一个韵母作为其韵脚，例如头(tou)韵脚为u，与图(tu)是押韵的

```sql
create view wuyan_ci_full as
select a.*,replace(substring(a.ch,patindex('%[a-z]%',a.ch)-1, len(a.ch)),'_','') cixing,
replace(substring(a.ch,1,charindex('_',a.ch)),'_','') ci,
substring(reverse(rtrim(b.py)),patindex('%[aeiou]%',reverse(rtrim(b.py))),1) py 
from wuyan_cipin a, xhzd b
where b.zi=substring(rtrim(replace(substring(a.ch,1,charindex('_',a.ch)),'_','')),len(rtrim(replace(substring(a.ch,1,charindex('_',a.ch)),'_',''))),1)

select * from wuyan_ci_full
```

### 模板生成

​    分词后的结果样例如下所示。

```sql
高枕_n 松间石_n ，_w 如_v 侬_g 未_d 易_a 知_v 。
```

​    从中我们可以找到“模板”，比如上面这个句子就是以

```sql
n n 23
v g d a v 11111
```

的方式生成的。我们利用下面的程序，得到五言诗的模板

```sql
declare @str nvarchar(200)

if object_id('tempdb.dbo.#array') is not null  --判断临时表是否已经存在
   drop table #array
create table #array(ch nvarchar(20))

--全部改成qiyan即可创建七言的词频库
declare cur scroll cursor for 
    select fenci from wuyan_fenci
open cur
fetch first from cur into @str
while @@FETCH_STATUS = 0
begin
    insert into #array(ch) select ltrim(dbo.get_letter(value))+ltrim(dbo.get_hanzi_num(value)) from 
    string_split(replace(replace(@str,'。','，'),'_w',''),'，')
	fetch next from cur into @str
end
close cur
deallocate cur

select ch, count(*) cnt into wuyan_muban
from #array
where ch is not null and ch<>' '
group by ch 
order by cnt desc
update wuyan_muban set ch=replace(ch,'0','')
drop table #array
```

下面的程序可以找出排名前10的模板

```sql
select top 10 * from wuyan_muban 
order by cnt desc
```

结果为

ch	cnt
id 5	82910
n v n 212	20683
n n 23	14030
id 5	13932
v n 23	12920
n id 23	11480
n np 23	8689
n d v 212	8447
n n 32	7675
v v n 212	6712

​    注意此处id是习语，所以除去它之外的模板，最常用的是n v n 212

## 三、功能设计

此部分我们仍然采用五言作为例子，七言只需要修改部分代码即可。

### 随机生成诗歌

​    随机生成诗歌，要满足两条。第一个是要押韵，每一句的韵脚都要统一（但事实上好像只需要13,24统一就行），第二是要选用不同的模板。

```sql
create proc wuyan_suiji(
    @m int
)
as
begin
    declare @temp nvarchar(30),@num nvarchar(6),@ty nvarchar(10)
    declare @str nvarchar(2),@s nvarchar(10)=' ',@res nvarchar(200)=' '
    declare @i int=0, @n int, @step int = 1, @res_out nvarchar(400)='', @yunjiao nvarchar(100)
    while @step<=@m
    begin
        select @res = ' ',@i=0
        select top 1 @temp = ch from wuyan_muban
            where cnt>20
            order by NEWID()
        set @num = substring(@temp,patindex('%[0-9]%',@temp),len(@temp))
	    set @num = replace(replace(@num,'_',''),'-','')
        set @ty = substring(@temp, 1, patindex('%[0-9]%',@temp)-1)
        declare cur scroll cursor for
            select value from string_split(@ty, ' ')
        open cur
        fetch first from cur into @str
        while @@FETCH_STATUS=0
        begin
            set @i = @i+1
	        set @n = cast(substring(@num,@i,1) as int)
			if @i=len(rtrim(ltrim(@num))) and @step>1
		    begin
		        select top 1 @s=ci from wuyan_ci_full
	            where cixing=@str and len(rtrim(ltrim(ci)))=@n
			    and cnt>100 and py=@yunjiao
		        order by NEWID()
			    if len(ltrim(rtrim(@s)))<>@n
		        begin
		            select top 1 @s=ci from wuyan_ci_full
	                where cixing=@str and len(rtrim(ltrim(ci)))=@n
			        and cnt>1 and py=@yunjiao
		            order by NEWID()
		        end
		    end
	        else
		    begin
	            select top 1 @s=ci from wuyan_ci_full
	            where cixing=@str and len(rtrim(ltrim(ci)))=@n
			    and cnt>100
		        order by NEWID()
			    if len(ltrim(rtrim(@s)))<>@n
		        begin
		            select top 1 @s=ci from wuyan_ci_full
	                where cixing=@str and len(rtrim(ltrim(ci)))=@n
			        and cnt>1
		            order by NEWID()
		        end
		    end
 	        set @res=concat(@res,@s)
	        fetch next from cur into @str
        end
        close cur 
	    deallocate global cur
	    if len(ltrim(rtrim(@res)))=5
	    begin
		    if @step=1
			begin
			    select @yunjiao=substring(reverse(rtrim(py)),patindex('%[aeiou]%',reverse(rtrim(py))),1) from xhzd where zi=substring(rtrim(@res),len(rtrim(@res)),1)
			end
            set @res_out=concat(@res_out,@res)
	        set @step = @step+1
	    end
    end
	select @res_out
end

--调用存储过程
exec wuyan_suiji @m=4
```

修改@m并运行存储过程就可以得到想要的结果。我们以@m=4,作为样例，得到结果如下

```sql
 林间先更争 
 至道老收声 
 择奇性刚野 
 何刻归斯文
```

### 藏头诗

​    藏头诗的思路较为轻松，每一句的第一个字和所给词的相应位置相同即可。此时为了计算速度，我们抛弃掉了随机模板，采用n v n 212的模板生成（若找不到，则变为n v n 122）

```sql
--生成存储过程
create proc wuyan_cangtou (
    @head nvarchar(100)
)
as
begin
	declare @temp nvarchar(30),@num nvarchar(6),@ty nvarchar(10)
	declare @str nvarchar(2),@s nvarchar(10),@res_tmp nvarchar(200)=' '
	declare @i int=0, @n int, @step int = 1
	declare @yunjiao nvarchar(10),@res nvarchar(200)
	set @res = ''
	while @step<=len(@head)
	begin
		select @res_tmp = ''
		select top 1 @res_tmp = ci from wuyan_ci_full
		where substring(ci,1,1)=substring(@head,@step,1) and cixing='n' and len(ltrim(rtrim(ci)))=2
		and cnt>1
		order by NEWID()
		set @res_tmp = @res_tmp+(select top 1 ci from wuyan_ci_full
		where cixing='v' and len(ltrim(rtrim(ci)))=1 and cnt>1
		order by NEWID())
		if @step=1
		begin
			set @res_tmp = @res_tmp +(select top 1 ci from wuyan_ci_full
			where cixing='n' and len(ltrim(rtrim(ci)))=2 and cnt>1
			order by NEWID())
			select @yunjiao=substring(reverse(rtrim(py)),patindex('%[aeiou]%',reverse(rtrim(py))),1) from xhzd where zi=substring(rtrim(@res_tmp),len(rtrim(@res_tmp)),1)
		end
		else
		begin
			set @res_tmp = @res_tmp +(select top 1 ci from wuyan_ci_full
			where cixing='n' and len(ltrim(rtrim(ci)))=2 and py=@yunjiao and cnt>1
			order by NEWID())
		end
		if len(rtrim(ltrim(@res_tmp)))<5
		begin
			select @res_tmp = ''
			select top 1 @res_tmp = ci from wuyan_ci_full
			where substring(ci,1,1)=substring(@head,@step,1) and cixing='n' and len(ltrim(rtrim(ci)))=1
			and cnt>1
			order by NEWID()
			set @res_tmp = @res_tmp+(select top 1 ci from wuyan_ci_full
				where cixing='v' and len(ltrim(rtrim(ci)))=2 and cnt>2
				order by NEWID())
			if @step=1
			begin
				set @res_tmp = @res_tmp +(select top 1 ci from wuyan_ci_full
				where cixing='n' and len(ltrim(rtrim(ci)))=2 and cnt>2
				order by NEWID())
				select @yunjiao=substring(reverse(rtrim(py)),patindex('%[aeiou]%',reverse(rtrim(py))),1) from xhzd where zi=substring(rtrim(@res_tmp),len(rtrim(@res_tmp)),1)
			end
			else
			begin
				set @res_tmp = @res_tmp +(select top 1 ci from wuyan_ci_full
				where cixing='n' and len(ltrim(rtrim(ci)))=2 and py=@yunjiao and cnt>2
				order by NEWID())
			end
		end
		set @res = concat(@res,' '+@res_tmp)
		set @step = @step+1
	end
	select @res
end
```

我们以不忘初心为头，运行

```sql
--调用
--declare @head nvarchar(10)='不忘初心'
--exec wuyan_cangtou @head
```

可以得到

```sql
 -- eg1
 不保寿暗空 
 忘物棘灵童 
 初路颅腷膊 
 心专擎坡坨
 -- eg2
 不受降雄图 
 忘物节客伫 
 初度尔爰居 
 心镜乖忆父
```

再来个牢记使命

```sql
 牢栅夕句漏 
 记意驷今俗 
 使令号道曲 
 命子将紫绶
```

整体运行一下

```sql
 不焚燎世勋 
 忘物霍殿楼 
 初辞了噪庐 
 心泉宰钩距 
 牢笼途衣垢 
 记就硕山隩 
 使客翮丹藕 
 命子瀁苍龟
```

### 飞花令

​    现代改进的飞花令一般是给定字词背诵出相应的诗句。利用SQL很容易筛选出相应的诗句，但我们需要将contents中含有给定字词的句子提取出来，需要一点功夫。我们可以建立一个函数来实现这一目标。其中`@i`表示`@str`中第一次出现`@a`的位置，`@j`表示`@a`上一个句号出现的位置，`@k`表示`@a`下一个句号出现的位置。

```sql
create function process_feihualing(@str nvarchar(200), @a nvarchar(10))
returns nvarchar(100)
as
begin
    declare @res nvarchar(100), @i int, @j int, @k int
	set @i = patindex('%'+@a+'%',@str)
	if patindex('%。%',reverse(substring(@str,1,@i)))=0
	begin
	    set @j = 1
	end
	else
	begin
	    set @j = 2+@i-patindex('%。%',reverse(substring(@str,1,@i)))
	end
	set @k = @i+patindex('%[。.]%',substring(@str,@i,len(@str)))-1
	return substring(@str,@j,@k-@j+1)
end

select top 10 *,dbo.process_feihualing(contents,'春') from shige_full
where contents like '%春%'
```

然后我们再建立存储过程

```sql
create proc feihualing(
    @ling nvarchar(10)
)
as 
begin
    select top 1 *, dbo.process_feihualing(contents,'%'+@ling+'%') juzi
    from shige_full
    where patindex('%'+@ling+'%',contents)>0
    order by NEWID()
end
```

即可实现我们的目标

## 四、服务器搭建*

​    这一part才是最难的一部分。本次小程序搭建框架为python+flask+nginx+gunicorn

​    在Ubuntu中，由于技术限制，我们选择了安装sql server。遇到的难点有：1. 安装sql server要求运行内存大于2G，但我们资金不够。所以需要利用python进入文件中修改二进制文件

```
1）cd /opt/mssql/bin/  # 进入sqlserver 目录 

2）mv sqlservr sqlservr.old  # 保存备份文件 

3）python # 使用python修改内存限制的二进制文件

 >>>oldfile = open("sqlservr.old", "rb").read()

>>>newfile = oldfile.replace("\x00\x94\x35\x77", "\x00\x80\x84\x1e")

>>>open("sqlservr", "wb").write(newfile)

>>>exit()

4） sudo /opt/mssql/bin/mssql-conf setup   进行sqlserver配置
```

其他文件的部署，我们参考了以下文章

https://blog.csdn.net/qq_40423339/article/details/86606308

其中遇到的比较困难的一步是nginx静态网页不支持post请求，需修改配置文件使得其重定向。参考解决办法

https://www.w3h5.com/post/419.html
完成以上Ubuntu系统下的配置后，需利用python搭建框架，示例如下

```python
from flask import Flask, request, json
import pymssql

app = Flask(__name__)
@app.route('/wuyan_suiji', methods=['POST'])
def wuyan_suiji():
    sql = """
    declare @m int = %s
    exec wuyan_suiji @m
    """
    length = int(json.loads(request.values.get("length")))
    conn = pymssql.connect(server='121.40.103.31',user='sa',password='Swufe@2021',database='project')
    cursor = conn.cursor()
    cursor.execute(sql, length)
    res_list = cursor.fetchall()
    result = res_list[0][0]
    cursor.close()
    conn.close()
    res = {'data':result}
    return json.dumps(res, ensure_ascii=False)
```

将该python文件放置于Ubuntu python虚拟环境的文件夹中(我们的位置是/root/venv)，然后运行

```
cd /root/venv
gunicorn -w 2 -b 127.0.0.1:5000 cangtou:app
```

然后即可利用微信小程序中的wx.request()进行传参，最终得到结果
## 五、微信小程序开发
​  

## 六、总结

​    