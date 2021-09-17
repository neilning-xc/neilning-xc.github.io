---
title: Laravel测试数据库的另一种替代方案-SQLite
date: 2018-07-13 17:01:41
tags: ["Laravel", "PHP"]
---

### Laravel测试数据库
在测试方面，`Laravel`内置使用`PHPUnit`提供了非常方便的解决方案。而对于数据库增删改查的测试，要解决的一个很重要的问题就是如何在测试完成之后，恢复数据库的原貌，例如要测试一个用户注册的方法，需要插入一条用户记录到数据库，但是测试完成之后，我们并不想让这条测试用例保存在数据库里。为了解决这个问题，`Laravel`提供了非常方便的方案：
- 使用迁移：`DatabaseMigrations`
- 使用事务：`DatabaseTransactions`

参考资料：
> [https://laravel.com/docs/5.3/database-testing#resetting-the-database-after-each-test](https://laravel.com/docs/5.3/database-testing#resetting-the-database-after-each-test)

### 另外一种解决方案:`SQLite`的内存数据库`:memory:`
`Laravel`提供的两种解决方案，仍然对数据库进行了读写操作，某些时候你可能并不想这样（例如多人共享一个线上开发数据库），此时，还可以用一种更为优雅的方式：`SQLite`，逻辑其实也非常简单：就是在跑测试用例的时候，将数据库的连接替换为`SQLite`。

### 使用示例
例如我们有以下测试类（该事例并不具有代表性，仅用于说明问题，并假设本机已经安装`SQLite`）：
```
class HomePageTest extends TestCase {
    
    public function testHomePage() 
    {
        // 创建一个测试用户，并保存
        $user = factory(App\User::class)->create();
        $this->actingAs($user)->visit('/home')->see('Dashboard');
    }
}
```
1. 首先在`Laravel`的数据库配置文件，即`config/database.php`的`connections`数组中添加一个新的数据库连接
```
'sqlite' => [
    'driver' => 'sqlite',
    'database' => ':memory:',
    'prefix' => '',
],
```
这里一个非常重要的参数就是`'database' => ':memory:'`，`:memory:`数据库是`SQLite`中内置的一个内存数据库，每次运行测试用例的时候都会在内存中创建一个新的数据库，并在测试完成关闭数据库连接后，自动清除掉，具有良好的隔离性，又因为是在内存中的，所以速度也很快，这些特性对于测试非常方便，这也是我们选用`SQLite`数据库作为测试库的一个非常重要的原因。有关详细解释，[点击这里](https://www.sqlite.org/inmemorydb.html)。

2. 然后需要修改`PHPUnit`的配置文件，在`phpunit.xml`中，将数据库连接改为刚刚定义的`SQLite`连接
```
<php>
    <env name="APP_ENV" value="testing"/>
    <env name="CACHE_DRIVER" value="array"/>
    <env name="SESSION_DRIVER" value="array"/>
    <env name="QUEUE_DRIVER" value="sync"/>
    
    <!-- 将数据库连接改为刚刚定义的SQLite连接 -->
    <env name="DB_CONNECTION" value="sqlite" />
</php>
```
这会覆盖`.env`中定义的数据库连接`DB_CONNECTION=mysql`，并告诉框架在运行测试的时候，用`sqlite`连接代替`mysql`连接。

3. 最后需要在运行测试用例之前执行数据库迁移，在测试类的基类，即`TestCase.php`中添加`setUp`方法
```
public function setUp()
{
    parent::setUp();
    // 执行数据库迁移 
    $this->artisan('migrate');
}
```
这样在每次执行测试用例之前，都会向`SQLite`的`:memory:`数据库中执行所有的迁移，生成业务逻辑所需的数据表。

### 方案优缺点
- 该方案关键点在于使用`SQLite`内置的一个内存数据库`:memory:`，因此速度比较快，有很好的隔离性，也不会对我们的开发数据库有任何的影响。

- 当然该方案也有缺点，假如项目的数据库庞大，有大量的数据表或者迁移文件，会消耗大量内存，当运行测试的时候可能会由于内存不足，导致测试中断。此时需要为PHP分配合适的内存，在`php.ini`中修改配置`memory_limit = 128M`

> 参考资料

- [https://laravel.com/docs/5.3/database-testing#resetting-the-database-after-each-test](https://laravel.com/docs/5.3/database-testing#resetting-the-database-after-each-test)
- [http://blog.mauriziobonani.com/laravel-sql-memory-database-for-unit-tests/](http://blog.mauriziobonani.com/laravel-sql-memory-database-for-unit-tests/)
- [https://www.sqlite.org/inmemorydb.html](https://www.sqlite.org/inmemorydb.html)
- [https://laracasts.com/discuss/channels/testing/how-to-specify-a-testing-database-in-laravel-5](https://laracasts.com/discuss/channels/testing/how-to-specify-a-testing-database-in-laravel-5)
