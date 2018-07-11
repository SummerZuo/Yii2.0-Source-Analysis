## DI Container

[参考文档]: http://www.digpage.com/service_locator.html

### 1.介绍

​	DI Container: dependence injection，是IOC的一种设计模式。

​	DI的核心是把类所依赖的单元的实例化过程，放到类的外面去实现。



### 2.DI容器的数据结构

![DataOfDI](.\img\DataOfDI.png)



### 3.注册依赖

1. 注册依赖就是向_definitions、__params数组中注入定义、构造函数参数
2. 去除掉已经存在的_singletons

```php
// 
public function set($class, $definition = [], array $params = [])
{
    $this->_definitions[$class] = $this->normalizeDefinition($class, $definition);
    $this->_params[$class] = $params;
    unset($this->_singletons[$class]);
    return $this;
}
// 只维护一个单例
public function setSingleton($class, $definition = [], array $params = [])
{
    $this->_definitions[$class] = $this->normalizeDefinition($class, $definition);
    $this->_params[$class] = $params;
    $this->_singletons[$class] = null;
    return $this;
}
```



### 4.获取依赖

​	**依赖注入体现为构造函数注入**

```php
protected function getDependencies($class)
{
    if (isset($this->_reflections[$class])) {
        return [$this->_reflections[$class], $this->_dependencies[$class]];
    }

    $dependencies = [];
    $reflection = new ReflectionClass($class);

    $constructor = $reflection->getConstructor();
    if ($constructor !== null) {
        foreach ($constructor->getParameters() as $param) {
            if (version_compare(PHP_VERSION, '5.6.0', '>=') && $param->isVariadic()) {
                break;
            } elseif ($param->isDefaultValueAvailable()) {
                $dependencies[] = $param->getDefaultValue();
            } else {
                $c = $param->getClass();
                $dependencies[] = Instance::of($c === null ? null : $c->getName());
            }
        }
    }

    $this->_reflections[$class] = $reflection;
    $this->_dependencies[$class] = $dependencies;
    return [$reflection, $dependencies];
}
```



### 5.解析依赖

```php
protected function resolveDependencies($dependencies, $reflection = null)
{
    foreach ($dependencies as $index => $dependency) {
        if ($dependency instanceof Instance) {
            if ($dependency->id !== null) {
                $dependencies[$index] = $this->get($dependency->id);
            } elseif ($reflection !== null) {
                $name = $reflection->getConstructor()->getParameters()[$index]->getName();
                $class = $reflection->getName();
                throw new InvalidConfigException("Missing required parameter \"$name\" when instantiating \"$class\".");
            }
        }
    }

    return $dependencies;
}
```



### 6.创建实例

```php
protected function build($class, $params, $config)
{
    /* @var $reflection ReflectionClass */
    list($reflection, $dependencies) = $this->getDependencies($class);

    foreach ($params as $index => $param) {
        $dependencies[$index] = $param;
    }

    $dependencies = $this->resolveDependencies($dependencies, $reflection);
    if (!$reflection->isInstantiable()) {
        throw new NotInstantiableException($reflection->name);
    }
    if (empty($config)) {
        return $reflection->newInstanceArgs($dependencies);
    }

    $config = $this->resolveDependencies($config);

    if (!empty($dependencies) && $reflection->implementsInterface('yii\base\Configurable')) 		{
        // set $config as the last parameter (existing one will be overwritten)
        $dependencies[count($dependencies) - 1] = $config;
        return $reflection->newInstanceArgs($dependencies);
    }

    $object = $reflection->newInstanceArgs($dependencies);
    foreach ($config as $name => $value) {
        $object->$name = $value;
    }

    return $object;
}
```



### 7.实例获取

​		**$class** ：表示将要创建或者获取的对象。可以是一个类名、接口名、别名。

​		**$params**：用于这个要创建的对象的构造函数的参数。

​		**$config**：是一个配置数组，用于配置获取的对象。

```php
public function get($class, $params = [], $config = [])
{
    if (isset($this->_singletons[$class])) {
    // singleton
    return $this->_singletons[$class];
    } elseif (!isset($this->_definitions[$class])) {
    return $this->build($class, $params, $config);
    }

    $definition = $this->_definitions[$class];

    if (is_callable($definition, true)) {
        $params = $this->resolveDependencies($this->mergeParams($class, $params));
        $object = call_user_func($definition, $this, $params, $config);
    } elseif (is_array($definition)) {
        $concrete = $definition['class'];
        unset($definition['class']);

        $config = array_merge($definition, $config);
        $params = $this->mergeParams($class, $params);

        if ($concrete === $class) {
            $object = $this->build($class, $params, $config);
        } else {
            $object = $this->get($concrete, $params, $config);
        }
    } elseif (is_object($definition)) {
    	return $this->_singletons[$class] = $definition;
    } else {
    throw new InvalidConfigException('Unexpected object definition type: ' . gettype($definition));
    }

    if (array_key_exists($class, $this->_singletons)) {
        // singleton
        $this->_singletons[$class] = $object;
    }

    return $object;
}
```



### 8.实际应用

```php
Yii::$container->set('test','yii\db\Connection');
var_dump(Yii::$container->get('test'));
var_dump(Yii::$container->get('yii\web\Request'));
```

