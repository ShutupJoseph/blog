# 第八节：MySQL锁机制

## 一、悲观锁与乐观锁
在电商秒杀等高并发场景中，多个用户同时秒杀同一件商品，如果不加锁，那么可能会出现数据异常或者超卖的现象，为此针对这些数据，需要加入锁机制。

常见的锁机制有**悲观锁**和**乐观锁**，他们的原理分别如下：

1. 悲观锁：悲观锁是数据库层面实现的。之所以成为悲观锁，是由于其假定数据操作过程种会产生冲突，那么在一次的数据操作过程种，会将数据处于锁定的状态，直至操作结束。在`MySQL`数据库中，使用`InnoDB`引擎的锁机制是行级数据锁定，而`MyISAM`引擎是表级锁定，一般情况而言，行级锁机制的效率比表级的锁机制效率更高。
2. 乐观锁：乐观锁是代码层面实现的。之所以成为乐观锁，是由于其假定数据操作过程中不会产生冲突，在操作数据时，不会将数据处于锁定状态，而是通过先判断该数据是否有冲突，如果没有冲突则正常执行，如果有冲突则停止访问，或者等待一段时间再访问。

## 二、悲观锁
### with_for_update：
在`sqlalchemy`中，可以使用`with_for_update`方法来实现悲观锁。示例代码如下：

```python
# 开启事务
async with session.begin():
    result = await session.execute(select(Seckill.stock).where(Seckill.id==seckill_id).with_for_update())
    seckill = result.scalar()
    seckill.stock -= 1
```

以上代码会输出类似以下的`sql`语句：

```plsql
SELECT seckill.max_sk_count 
FROM seckill 
WHERE seckill.id = %s FOR UPDATE
```

`with_for_update`方法的参数及其意义如下：

+ `nowait`：默认是`False`。<font style="color:rgb(26, 32, 41);">如果设置为</font>`<font style="color:rgb(26, 32, 41);">True</font>`<font style="color:rgb(26, 32, 41);">，则当请求的行被其他事务锁定时，查询将不会等待，而是立即抛出一个异常。</font>
+ `<font style="color:rgb(26, 32, 41);">read</font>`<font style="color:rgb(26, 32, 41);">：默认值是</font>`<font style="color:rgb(26, 32, 41);">False</font>`<font style="color:rgb(26, 32, 41);">。如果设置为</font>`<font style="color:rgb(26, 32, 41);">True</font>`<font style="color:rgb(26, 32, 41);">，则查询将使用读锁而不是写锁。这通常用于避免锁定不必要的行，只锁定那些实际需要更新的行。</font>
+ `<font style="color:rgb(26, 32, 41);">of</font>`<font style="color:rgb(26, 32, 41);">：指定要加锁的表或列。可以使用此参数来更精确地控制锁的范围。</font>
+ `<font style="color:rgb(26, 32, 41);">skip_locked</font>`<font style="color:rgb(26, 32, 41);">：默认是</font>`<font style="color:rgb(26, 32, 41);">False</font>`<font style="color:rgb(26, 32, 41);">。如果设置为</font>`<font style="color:rgb(26, 32, 41);">True</font>`<font style="color:rgb(26, 32, 41);">，则查询将跳过被其他事务锁定的行，只返回未被锁定的行。</font>
+ `<font style="color:rgb(26, 32, 41);">key_share</font>`<font style="color:rgb(26, 32, 41);">：默认是</font>`<font style="color:rgb(26, 32, 41);">False</font>`<font style="color:rgb(26, 32, 41);">。如果设置为</font>`<font style="color:rgb(26, 32, 41);">True</font>`<font style="color:rgb(26, 32, 41);">，则查询将使用键共享锁（key share lock），这是一种特殊的读锁，允许多个事务读取相同的行，但只允许一个事务更新这些行。</font>

### <font style="color:rgb(26, 32, 41);">读锁和写锁</font>
<font style="color:rgb(26, 32, 41);">在数据库中，锁是用来控制并发访问共享资源（如数据库表）的一种机制。读锁和写锁是两种基本的锁类型，它们定义了如何对数据进行访问和修改。</font>

#### <font style="color:rgb(26, 32, 41);">读锁（Read Lock）</font>
<font style="color:rgb(26, 32, 41);">读锁（也称为共享锁）是一种用于防止其他事务对数据进行写操作的锁。当一个事务对数据加上了读锁后，其他事务仍然可以读取这些数据，但不能修改或删除这些数据，直到读锁被释放。</font>

<font style="color:rgb(26, 32, 41);">读锁通常用于以下情况：</font>

+ <font style="color:rgb(26, 32, 41);">允许多个事务同时读取数据，但阻止任何事务修改数据。</font>
+ <font style="color:rgb(26, 32, 41);">防止其他事务修改数据，从而避免读取到未提交的数据（脏读）。</font>

#### <font style="color:rgb(26, 32, 41);">写锁（Write Lock）</font>
<font style="color:rgb(26, 32, 41);">写锁（也称为排他锁或独占锁）是一种用于防止其他事务对数据进行读或写操作的锁。当一个事务对数据加上了写锁后，其他事务不能读取或修改这些数据，直到写锁被释放。</font>

<font style="color:rgb(26, 32, 41);">写锁通常用于以下情况：</font>

+ <font style="color:rgb(26, 32, 41);">允许一个事务修改或删除数据，但阻止其他事务读取或修改数据。</font>
+ <font style="color:rgb(26, 32, 41);">防止其他事务读取或修改数据，从而避免读取到未提交的数据（脏读）。</font>

<font style="color:rgb(26, 32, 41);">在SQLAlchemy中，你可以使用</font>`<font style="color:rgb(26, 32, 41);">with_for_update()</font>`<font style="color:rgb(26, 32, 41);">方法来对查询结果集加锁。如果你设置</font>`<font style="color:rgb(26, 32, 41);">read=True</font>`<font style="color:rgb(26, 32, 41);">，则使用读锁；如果你不设置或设置为</font>`<font style="color:rgb(26, 32, 41);">False</font>`<font style="color:rgb(26, 32, 41);">，则使用写锁。</font>

<font style="color:rgb(26, 32, 41);">例如，如果你想使用读锁来防止其他事务修改数据，你可以这样使用</font>`<font style="color:rgb(26, 32, 41);">with_for_update()</font>`<font style="color:rgb(26, 32, 41);">：</font>复制

```python
stmt = select(MyModel).where(MyModel.id == some_id).with_for_update(read=True)
```

<font style="color:rgb(26, 32, 41);">而如果你想使用写锁来防止其他事务读取或修改数据，你可以这样使用</font>`<font style="color:rgb(26, 32, 41);">with_for_update()</font>`<font style="color:rgb(26, 32, 41);">：</font>复制

```python
stmt = select(MyModel).where(MyModel.id == some_id).with_for_update()  # 默认使用写锁
```

## 三、乐观锁
乐观锁是在代码层面实现的，一般是在需要加锁的表上加一个`version`字段，用来记录当前的版本号，在操作数据之前，先判断当前版本是否正确，如果正确，则可以正常操作，否则应该要继续等待。在`sqlalchemy`中，可以通过以下方式来实现乐观锁：

```python
class Seckill(Base, SnowFlakeIdModel, SerializerMixin):
    __tablename__ = 'seckill'
    serialize_only = ('id', 'sk_price', 'start_time', 'end_time', 'create_time', 'max_sk_count', 'sk_per_max_count', 'commodity')
    sk_price = Column(DECIMAL(10, 2), comment='秒杀价')
    start_time = Column(DateTime, comment='秒杀开始时间')
    end_time = Column(DateTime, comment='秒杀结束时间')
    create_time = Column(DateTime, default=datetime.now)
    max_sk_count = Column(Integer, comment='秒杀数量')
    stock = Column(Integer)
    sk_per_max_count = Column(Integer, comment='每人最多秒杀数量')
    version_id = Column(String(100), nullable=False)

    commodity_id = Column(BigInteger, ForeignKey('commodity.id'))
    commodity = relationship(Commodity, backref=backref('seckills'), lazy='joined')

    __mapper_args__ = {
        "version_id_col": version_id,
        "version_id_generator": lambda version: uuid.uuid4().hex
    }
```

要使用乐观锁去修改数据，则应该先查找出对应的数据，然后再进行更新操作。示例代码如下：

```python
async with session.begin():
    result = await session.execute(select(Seckill).where(Seckill.id==seckill_id))
    seckill = result.scalar()
    seckill.stock -= 1
```

以上代码在执行完后，输出以下类似的`sql`语句。示例代码如下：

```plsql
UPDATE seckill 
SET stock=%s, version_id=%s 
WHERE seckill.id = %s AND seckill.version_id = %s
(9, 'c87ae04cde9144dc854d2b02db0f672c', 1824447039207899136, '16c00af3d9b443a580fad8d0d2f56ddf')
```



> 原文: <https://www.yuque.com/hynever/shtqfp/bg6lci8mpulycgh9>