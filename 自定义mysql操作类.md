##### Author : GZH
##### Date   : 2022-3-7

## 自定义数据库操作类

### 准备

```
    import pymysql # 需要导入的包
    类名 : Model
    实例中的属性:
            table_name = None # 表名
            self.link = pymysql.connect(host=config.HOST, user=config.USER, password=config.PASSWD, db=config.DBNAME,charset='utf8') #数据库连接对象
            self.cursor = self.link.cursor(pymysql.cursors.DictCursor) #游标对象
            pk = 'id' #主键字段名
            fields=[] #表中所有字段名
    类中魔术方法:
            _init__(table,config=config)#构造方法，实现数据库连接等初始化工作
                参数: table被操作的数据表名
                参数: config 导入的数据库连接配置信息
            __del__()#析构方法，完成关闭数据库操作
    
    config.py
    重要参数的配置信息:
        HOST = 'localhost'  # 主机名
        USER = 'root'  # 用户名
        PASSWD = 'yujie113'  # 密码
        DBNAME = 'mydb'  # 数据库名
        PORT = 3306  # 端口
```

### Model类中的方法:

```
    1.def __init__(self, table, config=config)
    # 构造方法 实现数据库连接初始化
    
    2.def __loadFields(self)
    # 内部私有方法,负责加载当前表中的主键名和其他字段名信息
    
    3.def find(self, id)
    # 获取指定id号的单条数据
    
    4.def findall(self)
    # 获取表中所有信息
    
    5.def add(self, data={})
    # 添加数据方法,通过传入字典信息完成信息的添加
    
    6.def update(self, data={})
    # 修改数据方法,通过传入字典信息完成信息的修改
    
    7.def delete(self, id)
    # 删除指定id(主键)号的单条数据
    
    8.def select(self, where=[], order=None, limit=None)
    # 更准确的查询
    
    9.def __def__(self):
    # 析构方法 做一些关闭操作 垃圾回收机制
```

### mysql常用命令:

```
1.查询指定id号的数据
select * from game where id=1

2.查询表中所有数据
select * from game

3.向表中添加一条数据
insert into game(title,type,views,pic,time) values('赛博朋克','动作冒险','999','www.saibo.com','2022-3-7')

4.修改一条数据
update game set title='赛博朋克',type='动作冒险',views='999',pic='www.saibo.com',time='2022-3-7' where id="312"

5.删除一条数据(需要指定id)
delete from game where id=313

6.详细查询
select * from game where views>500 and type="动作冒险" order by views desc limit 99
desc : 降序
```

`文件结构`

```
-code         |---- __init_.py
    |         |
    --db-------config.py
    |         |  
    |         |___  model.py
    |
    |
     --test.py

```

### 完整代码:

#### model.py

```python
import pymysql
from db import config


class Model:
    '''单表信息操作类'''

    def __init__(self, table, config=config):
        '''构造方法 实现数据库连接初始化'''
        try:
            self.table_name = table
            self.link = pymysql.connect(host=config.HOST, user=config.USER, password=config.PASSWD, db=config.DBNAME,
                                        charset='utf8')
            self.cursor = self.link.cursor(pymysql.cursors.DictCursor)
            self.pk = 'id'  # 表的主键字段名
            self.fields = []  # 当前表中的字段列表名
            self.__loadFields()  # 调用私有方法初始化表字段信息
        except Exception as e:
            print("数据库初始化失败:", e)

    def __loadFields(self):
        '''内部私有方法,负责加载当前表中的主键名和其他字段名信息'''
        # 查询当前表结构信息
        sql = f'SHOW COLUMNS FROM {self.table_name}'
        self.cursor.execute(sql)
        dlist = self.cursor.fetchall()
        # 循环表中的每个字段信息
        for v in dlist:
            # 将每个字段名添加到属性fields中
            self.fields.append(v['Field'])
            # 判断当前字段是否是主键
            if v['Key'] == 'PRI':
                self.pk = v['Field']

    def find(self, id):
        '''获取指定id号的单条数据'''
        try:
            sql = f'select * from {self.table_name} where {self.pk}={id}'
            # print(sql)
            self.cursor.execute(sql)
            return self.cursor.fetchone()
        except Exception as e:
            print("SQL执行出现错误:", e)

    def findall(self):
        '''获取所有信息'''
        try:
            sql = 'select * from %s' % (self.table_name)
            print(sql)
            self.cursor.execute(sql)
            return self.cursor.fetchall()
        except Exception as e:
            print("SQL执行出现错误:", e)

    def add(self, data={}):
        '''添加数据方法,通过传入字典信息完成信息的添加'''
        try:
            # 组装sql语句的数据处理
            keys = []
            values = []
            for k, v in data.items():
                # 判断传进来的字段名是否合法
                if k in self.fields:
                    keys.append(k)
                    values.append(v)
            # str_key, str_value = ",".join(keys), ",".join(values)
            #             ','.join(['%s']*5)  -->  '%s','%s,%s,%s'
            sql = f'insert into {self.table_name}(%s) values(%s)' % (",".join(keys), ",".join(['%s'] * len(values)))
            # insert into game(title,type,views,pic,time) values(%s,%s,%s,%s,%s)
            self.cursor.execute(sql, tuple(values))
            self.link.commit()
            str = f'成功添加一条数据,数据的id是{self.cursor.lastrowid}'
            # return self.cursor.lastrowid # 返回自增的id值
            return str
        except Exception as e:
            print(e)
            return 0

    def update(self, data={}):
        '''修改数据方法,通过传入字典信息完成信息的修改'''
        try:
            # 组装sql语句的数据处理
            keys = []
            values = []
            for k, v in data.items():
                # 判断传进来的字段名是否合法
                if (k in self.fields) and (k != self.pk):
                    # 条件: str="小王"  '{}=%s'.format(str)  -->  小王=%s
                    # keys.append("{}=%s".format(k))
                    keys.append(f"{k}=%s")
                    values.append(v)
            sql = f'update {self.table_name} set {",".join(keys)} where {self.pk}="{data.get(self.pk)}"'
            #       update game set title=%s,type=%s,views=%s,pic=%s,time=%s where id="312"
            self.cursor.execute(sql, tuple(values))
            self.link.commit()
            # print(sql)
            str = f'成功修改一条数据,数据的id是{data.get(self.pk)}'
            # return self.cursor.lastrowid # 返回自增的id值
            return str
        except Exception as e:
            print(e)
            return 0

    def delete(self, id):
        '''删除指定id号的单条数据'''
        try:
            sql = f'delete from {self.table_name} where {self.pk}={id}'
            self.cursor.execute(sql)
            self.link.commit()  # 事物的提交
            # return self.cursor.rowcount # 返回影响行数
            print(sql)
            success = f'成功删掉id为{id}的数据'
            return success
        except Exception as e:
            print("SQL执行出现错误:", e)
            return 0  # 执行失败返回 0

    def select(self, where=[], order=None, limit=None):
        '''更准确的查询'''
        sql = f'select * from {self.table_name}'
        # 判断并拼装sql语句
        if isinstance(where, list) and len(where) > 0:
            sql += ' where ' + ' and '.join(where)
        if order is not None:
            sql += ' order by ' + f'{order}'
        if limit is not None:
            sql += ' limit ' + str(limit)
        print(sql)
        self.cursor.execute(sql)
        return self.cursor.fetchall()

    def __def__(self):
        '''析构方法 做一些关闭操作 垃圾回收机制'''
        if self.link != None:
            self.link.close()

```

#### test.py

```python

# 自定义model类的测试代码

from db.model import Model

# 程序入口判断
if __name__ == '__main__':
    mod = Model('game')
    # print(mod.find(id=1))
    # dlist = mod.findall()
    # for i in dlist:
    #     print(i)

    # 测试删除
    # print(mod.delete(313))

    data = {
        'id': 312,
        'title': '小猫钓鱼',
        'type': '益智',
        'views': '999',
        'pic': 'www.xiaomao.com',
        'time': '2022-3-7'
    }
    # print(mod.fields)
    # insert into game(title,type,views,pic,time) values('小猫钓鱼','益智','999','www.xiaomao.com','2022-3-7')
    # insert into game(title,type,views,pic,time) values(%s,%s,%s,%s,%s)

    # 测试添加数据
    # print(mod.add(data))

    # 测试更改数据
    # print(mod.update(data))

    # 测试查询
    # print(mod.select(where=['views > 60']))
    # print(mod.select(order='views desc',limit=3))
    dlist = mod.select(where=['views>500', 'type="动作冒险"'], order='views desc', limit=99)
    for i in dlist:
        print(i)

```

#### config.py

```python
# 数据库连接配置信息
HOST = 'localhost'  # 主机名
USER = 'root'  # 用户名
PASSWD = 'yujie113'  # 密码
DBNAME = 'mydb'  # 数据库名
PORT = 3306  # 端口

```

