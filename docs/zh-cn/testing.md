# 自动化测试

在 Hyperf 里测试默认通过 `phpunit` 来实现，但由于 Hyperf 是一个协程框架，所以默认的 `phpunit` 并不能很好的工作，因此我们提供了一个 `co-phpunit` 脚本来进行适配，您可直接调用脚本或者使用对应的 composer 命令来运行。自动化测试没有特定的组件，但是在 Hyperf 提供的骨架包里都会有对应实现。

```
composer require hyperf/testing
```

```json
"scripts": {
    "test": "co-phpunit -c phpunit.xml --colors=always"
},
```

## Bootstrap

Hyperf 提供了默认的 `bootstrap.php` 文件，它让用户在运行单元测试时，扫描并加载对应的库到内存里。

```php
<?php

declare(strict_types=1);

error_reporting(E_ALL);
date_default_timezone_set('Asia/Shanghai');

! defined('BASE_PATH') && define('BASE_PATH', dirname(__DIR__, 1));
! defined('SWOOLE_HOOK_FLAGS') && define('SWOOLE_HOOK_FLAGS', SWOOLE_HOOK_ALL);

Swoole\Runtime::enableCoroutine(true);

require BASE_PATH . '/vendor/autoload.php';

Hyperf\Di\ClassLoader::init();

$container = require BASE_PATH . '/config/container.php';

$container->get(Hyperf\Contract\ApplicationInterface::class);

```

运行单元测试

```
composer test
```

## 模拟 HTTP 请求

在开发接口时，我们通常需要一段自动化测试脚本来保证我们提供的接口按预期在运行，Hyperf 框架下提供了 `Hyperf\Testing\Client` 类，可以让您在不启动 Server 的情况下，模拟 HTTP 服务的请求：

```php
<?php
use Hyperf\Testing\Client;

$client = make(Client::class);

$result = $client->get('/');
```

因为 Hyperf 支持多端口配置，除了验证默认的端口接口外，如果验证其他端口的接口呢？

```php
<?php

use Hyperf\Testing\Client;

$client = make(Client::class, ['server' => 'adminHttp']);

$result = $client->json('/user/0',[
    'nickname' => 'Hyperf'
]);

```

默认情况下，框架使用 `JsonPacker`，会直接解析 `Body` 为 `array`，如果您直接返回 `string`，则需要设置对应 `Packer`

```php
<?php

use Hyperf\Testing\Client;
use Hyperf\Contract\PackerInterface;

$client = make(Client::class, [
    'packer' => new class() implements PackerInterface {
        public function pack($data): string
        {
            return $data;
        }

        public function unpack(string $data)
        {
            return $data;
        }
    },
]);

$result = $client->json('/user/0',[
    'nickname' => 'Hyperf'
]);
```

## 示例

让我们写个小 DEMO 来测试一下。

```php
<?php

declare(strict_types=1);

namespace HyperfTest\Cases;

use Hyperf\Testing\Client;
use PHPUnit\Framework\TestCase;

/**
 * @internal
 * @coversNothing
 */
class ExampleTest extends TestCase
{
    /**
     * @var Client
     */
    protected $client;

    public function __construct($name = null, array $data = [], $dataName = '')
    {
        parent::__construct($name, $data, $dataName);
        $this->client = make(Client::class);
    }

    public function testExample()
    {
        $this->assertTrue(true);

        $res = $this->client->get('/');

        $this->assertSame(0, $res['code']);
        $this->assertSame('Hello Hyperf.', $res['data']['message']);
        $this->assertSame('GET', $res['data']['method']);
        $this->assertSame('Hyperf', $res['data']['user']);

        $res = $this->client->get('/', ['user' => 'developer']);

        $this->assertSame(0, $res['code']);
        $this->assertSame('developer', $res['data']['user']);

        $res = $this->client->post('/', [
            'user' => 'developer',
        ]);
        $this->assertSame('Hello Hyperf.', $res['data']['message']);
        $this->assertSame('POST', $res['data']['method']);
        $this->assertSame('developer', $res['data']['user']);

        $res = $this->client->json('/', [
            'user' => 'developer',
        ]);
        $this->assertSame('Hello Hyperf.', $res['data']['message']);
        $this->assertSame('POST', $res['data']['method']);
        $this->assertSame('developer', $res['data']['user']);

        $res = $this->client->file('/', ['name' => 'file', 'file' => BASE_PATH . '/README.md']);

        $this->assertSame('Hello Hyperf.', $res['data']['message']);
        $this->assertSame('POST', $res['data']['method']);
        $this->assertSame('README.md', $res['data']['file']);
    }
}
```

## 调试代码

在 FPM 场景下，我们通常改完代码，然后打开浏览器访问对应接口，所以我们通常会需要两个函数 `dd` 和 `dump`，但 `Hyperf` 跑在 `CLI` 模式下，就算提供了这两个函数，也需要在 `CLI` 中重启 `Server`，然后再到浏览器中调用对应接口查看结果。这样其实并没有简化流程，反而更麻烦了。

接下来，我来介绍如何通过配合 `testing`，来快速调试代码，顺便完成单元测试。

假设我们在 `UserDao` 中实现了一个查询用户信息的函数
```php
namespace App\Service\Dao;

use App\Constants\ErrorCode;
use App\Exception\BusinessException;
use App\Model\User;

class UserDao extends Dao
{
    /**
     * @param $id
     * @param bool $throw
     * @return
     */
    public function first($id, $throw = true)
    {
        $model = User::query()->find($id);
        if ($throw && empty($model)) {
            throw new BusinessException(ErrorCode::USRE_NOT_EXIST);
        }
        return $model;
    }
}
```

那我们编写对应的单元测试

```php
namespace HyperfTest\Cases;

use HyperfTest\HttpTestCase;
use App\Service\Dao\UserDao;
/**
 * @internal
 * @coversNothing
 */
class UserTest extends HttpTestCase
{
    public function testUserDaoFirst()
    {
        $model = \Hyperf\Utils\ApplicationContext::getContainer()->get(UserDao::class)->first(1);

        var_dump($model);

        $this->assertSame(1, $model->id);
    }
}
```

然后执行我们的单测

```
composer test -- --filter=testUserDaoFirst
```

## 测试替身

`Gerard Meszaros` 在 `Meszaros2007` 中介绍了测试替身的概念：

有时候对 `被测系统(SUT)` 进行测试是很困难的，因为它依赖于其他无法在测试环境中使用的组件。这有可能是因为这些组件不可用，它们不会返回测试所需要的结果，或者执行它们会有不良副作用。在其他情况下，我们的测试策略要求对被测系统的内部行为有更多控制或更多可见性。

如果在编写测试时无法使用（或选择不使用）实际的依赖组件(DOC)，可以用测试替身来代替。测试替身不需要和真正的依赖组件有完全一样的的行为方式；他只需要提供和真正的组件同样的 API 即可，这样被测系统就会以为它是真正的组件！

下面展示分别通过构造函数注入依赖、通过 `@Inject` 注释注入依赖的测试替身

### 构造函数注入依赖的测试替身

```php
<?php

namespace App\Logic;

use App\Api\DemoApi;

class DemoLogic
{
    /**
     * @var DemoApi $demoApi
     */
    private $demoApi;

    public function __construct(DemoApi $demoApi)
    {
       $this->demoApi = $demoApi;
    }

    public function test()
    {
        $result = $this->demoApi->test();

        return $result;
    }
}
```

```php
<?php

namespace App\Api;

class DemoApi
{
    public function test()
    {
        return [
            'status' => 1
        ];
    }
}
```

```php
<?php

namespace HyperfTest\Cases;

use App\Api\DemoApi;
use App\Logic\DemoLogic;
use Hyperf\Di\Container;
use HyperfTest\HttpTestCase;
use Mockery;

class DemoLogicTest extends HttpTestCase
{
    public function tearDown()
    {
        Mockery::close();
    }

    public function testIndex()
    {
        $res = $this->getContainer()->get(DemoLogic::class)->test();

        $this->assertEquals(1, $res['status']);
    }

    /**
     * @return Container
     */
    protected function getContainer()
    {
        $container = Mockery::mock(Container::class);

        $apiStub = $this->createMock(DemoApi::class);

        $apiStub->method('test')->willReturn([
            'status' => 1,
        ]);

        $container->shouldReceive('get')->with(DemoLogic::class)->andReturn(new DemoLogic($apiStub));

        return $container;
    }
}
```

### 通过 Inject 注释注入依赖的测试替身

```php
<?php

namespace App\Logic;

use App\Api\DemoApi;
use Hyperf\Di\Annotation\Inject;

class DemoLogic
{
    /**
     * @var DemoApi $demoApi
     * @Inject()
     */
    private $demoApi;

    public function test()
    {
        $result = $this->demoApi->test();

        return $result;
    }
}
```

```php
<?php

namespace App\Api;

class DemoApi
{
    public function test()
    {
        return [
            'status' => 1
        ];
    }
}
```

```php
<?php

namespace HyperfTest\Cases;

use App\Api\DemoApi;
use App\Logic\DemoLogic;
use Hyperf\Di\Container;
use Hyperf\Utils\ApplicationContext;
use HyperfTest\HttpTestCase;
use Mockery;

class DemoLogicTest extends HttpTestCase
{
    public function tearDown()
    {
        Mockery::close();
    }

    public function testIndex()
    {
        $this->getContainer();

        $res = $this->getContainer()->get(DemoLogic::class)->test();

        $this->assertEquals(11, $res['status']);
    }

    /**
     * @return Container
     */
    protected function getContainer()
    {
        $container = ApplicationContext::getContainer();

        $apiStub = $this->createMock(DemoApi::class);

        $apiStub->method('test')->willReturn([
            'status' => 11
        ]);

        $container->getDefinitionSource()->addDefinition(DemoApi::class, function () use ($apiStub) {
            return $apiStub;
        });
        
        return $container;
    }
}
```

## 生成测试覆盖率

执行

`php bin/hyperf.php vendor:publish hyperf/testing`


- 修改 composer.json 在 autoload-dev 中增加 classmap 用来将PHPunit中检测 XDebug 扩展的地方修改为检测 SDebug

```json
    "autoload-dev": {
        ...
        "classmap": [
            "classmap/"
        ]
        ...
    },
```

- 添加 `coverage` `coverage-output` 命令

```json
    "scripts": {
        ...
        "coverage": "co-phpunit -c phpunit.xml --colors=always --coverage-html ./runtime/codeCoverage",
        "coverage-output": "co-phpunit -c phpunit.xml --colors=never --coverage-text",
        ...
    }

```

- 修改 phpunit.xml 去除 coverage 标签中 processUncoveredFiles="true" 否则会抛出异常

```
Fatal error: Cannot declare class App\Controller\AbstractController, because the name is already in use in /home/dickens7/worker/code/hyperf-skeleton/app/Controller/AbstractController.php on line 19

Call Stack:
    0.0005    1449288   1. {closure:/home/dickens7/worker/code/hyperf/src/testing/co-phpunit:8-68}() /home/dickens7/worker/code/hyperf/src/testing/co-phpunit:0
    0.0130    5341448   2. PHPUnit\TextUI\Command::main() /home/dickens7/worker/code/hyperf/src/testing/co-phpunit:65
    0.0130    5341560   3. PHPUnit\TextUI\Command->run() /home/dickens7/worker/code/hyperf-skeleton/vendor/phpunit/phpunit/src/TextUI/Command.php:96
    7.9843   84339040   4. PHPUnit\TextUI\TestRunner->run() /home/dickens7/worker/code/hyperf-skeleton/vendor/phpunit/phpunit/src/TextUI/Command.php:143
    8.0005   85614104   5. SebastianBergmann\CodeCoverage\Report\Html\Facade->process() /home/dickens7/worker/code/hyperf-skeleton/vendor/phpunit/phpunit/src/TextUI/TestRunner.php:737
    8.0006   85617328   6. SebastianBergmann\CodeCoverage\CodeCoverage->getReport() /home/dickens7/worker/code/hyperf-skeleton/vendor/phpunit/php-code-coverage/src/Report/Html/Facade.php:54
    8.0168   85673752   7. SebastianBergmann\CodeCoverage\Node\Builder->build() /home/dickens7/worker/code/hyperf-skeleton/vendor/phpunit/php-code-coverage/src/CodeCoverage.php:139
    8.0168   85673752   8. SebastianBergmann\CodeCoverage\CodeCoverage->getData() /home/dickens7/worker/code/hyperf-skeleton/vendor/phpunit/php-code-coverage/src/Node/Builder.php:44
    8.0168   85673752   9. SebastianBergmann\CodeCoverage\CodeCoverage->processUncoveredFilesFromFilter() /home/dickens7/worker/code/hyperf-skeleton/vendor/phpunit/php-code-coverage/src/CodeCoverage.php:167
    8.0874   85675928  10. include_once('/home/dickens7/worker/code/hyperf-skeleton/app/Controller/AbstractController.php') /home/dickens7/worker/code/hyperf-skeleton/vendor/phpunit/php-code-coverage/src/CodeCoverage.php:538
```

- 增加目录 <directory suffix=".php">./runtime/container/proxy</directory> 将代理类也加入测试覆盖范围

#### 修改后xml 示例如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" backupGlobals="false" backupStaticAttributes="false" bootstrap="./test/bootstrap.php" colors="true" convertErrorsToExceptions="true" convertNoticesToExceptions="true" convertWarningsToExceptions="true" processIsolation="false" stopOnFailure="false" xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.3/phpunit.xsd">
  <coverage>
    <include>
      <directory suffix=".php">./app</directory>
      <directory suffix=".php">./runtime/container/proxy</directory>
    </include>
  </coverage>
  <testsuites>
    <testsuite name="Tests">
      <directory suffix="Test.php">./test</directory>
    </testsuite>
  </testsuites>
</phpunit>
```