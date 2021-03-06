之前看到网上有一篇文章说Phalcon和PHP没有数据库连接池，而swoole本身提供了很好的数据库连接池。实际上这是一种误解，PHP自身早就实现了持久化的数据库连接。而Phalcon基于zephir写的数据库连接适配器，必然也是支持PHP自身实现的这种数据库连接池。Phalcon基于C语言写的ORM，配合PHP-FPM提供的多进程的PHP数据库连接池，将提供性能极为强悍并且健壮的多进程数据库连接池。苦逼的是我大Phalcon文档太烂，根本就没提到这方面的先天优势。根本不需要swoole，nginx与php-fpm的黄金搭档以及phalcon提供的高性能ORM，就能提供目前最好的PHP数据库连接池解决方案。本文将详述，有关PHP数据库连接池的相关技术。

### 第一：PHP官方文档有关持久化数据库连接的有关阐述

下图就是PHP官方网站，对于PHP持久化数据库连接问题的一些阐述。

![github](https://github.com/fupengfei058/article-collection/blob/master/doc/e1.png)

这里需要特别注意的一个问题是，由于脚本语言的特殊性，PHP官方并不建议在锁表或者事务操作的时候，使用PHP的持久化数据库连接。如图一所示，害怕数据库锁死

### 第二：PHP-FPM提供的进程复用机制给PHP提供了健壮的PHP数据库连接池

下图中所示的，就是PHP-FPM的配置文件。关于配置文件，PHPer都看得懂，我就不多说了。主要说说PHP-FPM带来的PHP数据库连接池的特色：

![github](https://github.com/fupengfei058/article-collection/blob/master/doc/e2.png)

1. PHP-FPM一个进程对应一个持久化的数据库连接；

2. 所有PHP-FPM进程所创造的持久化数据库连接，不能超过mysql的最大数据库连接数的，默认max_connections为150个连接。所以，不管是动态还是静态管理PHP-FPM进程，都需要注意PHP-FPM的进程数和mysql的最大连接数。

3. 如果一个PHP请求，请求了多个不同数据库的数据，创建了多个持久化的数据库连接。那么对应的PHP-FPM就会在一个进程内创建多个持久化的数据库连接。这里是一个比较坑的地方，所有PHPer都应该注意。

假设服务器有100个PHP-FPM的进程，单个请求向5个不同的数据库请求持久化连接的数据，那么服务器将会创建500个持久化的数据库连接。

### 第三：PHP实现数据库连接池的条件

![github](https://github.com/fupengfei058/article-collection/blob/master/doc/e3.png)

如上图中所示，只有在new Pdo（）时使用了Pdo::PERSISTENT=ture属性构造数据库连接时，这个连接才会成为持久化的数据库连接。但是问题来了，new Pdo（）时，绑定了数据库的地址、库名和用户名。也就是说，同一数据库的同一用户产生的数据库连接，才会复用这个数据库持久连接。

这是PHP数据库连接池的另外一个问题。

### 第四：Phalcon框架支持PHP内置的数据库连接池

虽然Phalcon官方文档并没有提及PDO的数据库连接池，但是翻看Phalcon的PDO类的源代码，就会发现Phalcon实际上是支持PDO的数据库连接池。如下图所示：

![github](https://github.com/fupengfei058/article-collection/blob/master/doc/e4.png)

不仅仅支持PHP的数据库连接池，而且他用C语言写的ORM，性能要比所有的PHP语言写的ORM要强很多。也就是说Phalcon的ORM及与PHP-FPM实现的数据库连接池，就是目前PHP所有框架中数据库连接池性能最强的。

yaf框架虽然路由性能比phalcon略好，但是由于没有ORM，在拉取PHP所写的ORM后，其ORM性能要比Phalcon内置的C语言所写的PHQL要差。

### 第五：Phalcon项目内数据库连接池的实现方式

Phalcon实现持久连接的方式很简单，如下图所示，配置文件里在注册服务时添加一个PDO的参数即可。所有通过这个请求的数据库连接，都将成为持久化的连接。

![github](https://github.com/fupengfei058/article-collection/blob/master/doc/e5.png)

但是由于，PHP数据库连接池的特殊性。最好通过注册不同的数据库连接服务，来区分不同的数据库连接。甚至针对高查询量的请求，创建带有专用数据库连接池的微服务，来实现与其他不需要数据库连接池业务的区分。

在如今微服务大行其道的情况下，PHP这种需要区别对待的数据库连接池，反而更容易使用了。

### 第六：关于需要锁表操作和事务操作的数据库连接池

PHP官方建议不要在锁表和事务操作的时候，使用数据库连接池。个人的看法是，一个项目中需要锁表和事务操作的地方，往往并不多，两者完全可以不走数据库连接池。程序员可以通过其他办法，尽量少写锁表操作。而对于并发较少的事务操作，用完连接就扔了，不走数据库连接池也没什么不可。

毕竟锁表和事务操作，真正需要用的并不是太多。实在不行的话，也可以用其他语言来写吧。诸如：Java，Go，C#，.NET Core等，通过他们搭建少量的微服务专门服务于锁表和事务操作作为补充，也未尝不可。

### 总结

如本文所示，通过nginx+php-fpm+phalcon就能通过性能极好的ORM搭建基于多进程的数据库连接池。根本不需要swoole扩展，就能实现php的数据库连接池。而且这种基于多进程的数据库连接池，比基于多线程的数据库连接池更加健壮和安全。

PHP的Phalcon框架，跟随PHP-FPM常驻内存，本身处理HTTP请求的性能极强。而且自带C语言所写的ORM，通过PHP-FPM实现基于多进程的数据库连接池，再搭配Nginx强大的HTTP处理能力，这真是完美的Web组合啊。

公司同事用最新版的Go在本地测试，跑了7000每秒的并发请求。鸟哥之前做的测试，Phalcon可是能跑到14000的RQS。哈哈，我大Phalcon真是吊的飞起呢。以后加上JIT和数据类型，PHP未来会更加强大。

链接：
http://www.jicker.cn/5641.html
