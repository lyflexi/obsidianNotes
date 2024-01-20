# MySQL悲观锁
在数据库中使用 for update 关键字可以实现悲观锁，我们在 Mapper 中添加 for update 即可对数据加锁，实现代码如下：
```xml
<!-- UserMapper.xml -->
<select id="selectByIdForUpdate" resultType="User">
    SELECT * FROM user WHERE id = #{id} FOR UPDATE
</select>
```

在 Service 中调用 Mapper 方法，即可获取到加锁的数据：
```java
@Transactional
public void updateWithPessimisticLock(int id, String name) {
    User user = userMapper.selectByIdForUpdate(id);
    if (user != null) {
        user.setName(name);
        userMapper.update(user);
    } else {
        throw new RuntimeException("数据不存在");
    }
}
```

# MySQL乐观锁

在 MyBatis 中，可以通过给表添加一个版本号字段来实现乐观锁。在 Mapper 中，使用 标签定义更新语句，同时使用 set 标签设置版本号的增量。
```xml
<!-- UserMapper.xml -->
<update id="updateWithOptimisticLock">
    UPDATE user SET
    name = #{name},
    version = version + 1
    WHERE id = #{id} AND version = #{version}
</update>
```

在 Service 中调用 Mapper 方法，需要传入更新数据的版本号。如果更新失败，说明数据已经被其他事务修改，自己就该放弃修改避免重复消费，具体实现代码如下：
```javascript
@Transactional
public void updateWithOptimisticLock(int id, String name, int version) {
    User user = userMapper.selectById(id);
    if (user != null) {
        user.setName(name);
        user.setVersion(version);
        int rows = userMapper.updateWithOptimisticLock(user);
        if (rows == 0) {
            throw new RuntimeException("数据已被其他事务修改");
        }
    } else {
        throw new RuntimeException("数据不存在");
    }
}
```