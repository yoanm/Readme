# PhpUnit configuration
## Configuration reference
```xml
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:noNamespaceSchemaLocation="http://schema.phpunit.de/4.8/phpunit.xsd"
  stopOnError="true"
  stopOnFailure="true"
  convertErrorsToExceptions="true"
  convertNoticesToExceptions="true"
  convertWarningsToExceptions="true"
  beStrictAboutOutputDuringTests="true"
  beStrictAboutChangesToGlobalState="true"
  beStrictAboutTestsThatDoNotTestAnything="true"
  backupGlobals="true"
  backupStaticAttributes="false"
  forceCoversAnnotation="true"
  processIsolation="false"
  checkForUnintentionallyCoveredCode="true"
  bootstrap="vendor/autoload.php"
  colors="true"
>
  <listeners>
        <listener class="StrictModeListener" file="tests/StrictModeListener.php"/>
  </listeners>

  <testsuites>
    <testsuite name="technical">
      <directory>tests/Technical/Unit/*</directory> <!-- launch unit before => faster than integration -->
      <directory>tests/Technical/Integration/*</directory>
    </testsuite>
    <testsuite name="functional"> <!-- defined functional after technical => longer than technical -->
      <directory>tests/Functional/*</directory>
    </testsuite>
  </testsuites>

  <filter>
    <whitelist>
      <directory>src</directory>
    </whitelist>
  </filter>
</phpunit>
```
## Requirements

  * `beStrictAboutChangesToGlobalState="true"`requires `backupGlobals="true"` in order to work

## Explanations

### Mandatory rules

  * [Early stop](../Tests.md#rules-early-stop)

    * `stopOnError="true"`
    * `stopOnFailure="true"`

  * [Strict mode](../Tests.md#rules-strict-mode)

    * [Strict mode - php errors](../Tests.md#rules-strict-mode-php-errors)

      * `convertErrorsToExceptions="true"`
      * `convertNoticesToExceptions="true"`
      * `convertWarningsToExceptions="true"`
    
    * [Strict mode - risky tests](../Tests.md#rules-strict-mode-risky-tests) (requires [`listener`](#listener))

      * `beStrictAboutOutputDuringTests="true"`
      * `beStrictAboutChangesToGlobalState="true"`
      * `beStrictAboutTestsThatDoNotTestAnything="true"`

  * [Risky test](../Tests.md#rules-risky-tests)

    * `beStrictAboutOutputDuringTests="true"` [Risky test - output](../Tests.md#rules-risky-tests-output)
    * `beStrictAboutChangesToGlobalState="true"` [Risky test - globals](../Tests.md#rules-risky-tests-manipulate-globals)
    * `beStrictAboutTestsThatDoNotTestAnything="true"` [Risky test - test nothing](../Tests.md#rules-risky-tests-test-nothing)
  
  * [Test isolation](../Tests.md#rules-tests-isolation)
    
    * `backupGlobals="true"` [test isolation - global](../Tests.md#rules-tests-isolation-globals)
      
      *Required by `beStrictAboutChangesToGlobalState="true"`*

    * `backupStaticAttributes="false"` : [test isolation - static](../Tests.md#rules-tests-isolation-static-class-member)
  
  * [Real coverage](../Tests.md#rules-real-coverage)
    
    * [Real coverage - risky tests](../Tests.md#rules-real-coverage-risky-tests)
    
      * `beStrictAboutOutputDuringTests="true"` (requires [`listener`](#listener))
      * `beStrictAboutChangesToGlobalState="true"`
      * `beStrictAboutTestsThatDoNotTestAnything="true"`
    
    * [Real coverage - no overflow](../Tests.md#rules-real-coverage-overflow)
      
      * `forceCoversAnnotation="true"`

        If no annotation, no coverage.

        Symply add the following as test class or test method comment
```
/**
 * @covers FULLY\QUALIFIED\NAMESPACE\TO\MyClass
 */
```

<a name="listener"></a>
 
### Optional rules (validated with [configuration reference](#configuration-reference))
  * [Test doc - tested class](../Tests.md##rules-test-documentation-tested-class-description) : by using `@covers`
      
    *In fact, required as configuration uses `forceCoversAnnotation="true"`*

  * [Test doc - tested class dependencies](../Tests.md#rules-test-documentation-tested-class-dependencies-description) by using `@uses`
  
    Also use `checkForUnintentionallyCoveredCode="true"`, to be sure sure that new dependencies will be forced to be documented. With [configuration reference](#configuration-reference), it will convert test into risky test, and risky test will be converted into failed test by [`listener`](#listener)
      
    Simply add the following as test class or test method comment : 
```
/**
 * @uses FULLY\QUALIFIED\NAMESPACE\TO\MyClassDependencyAClass
 * @uses FULLY\QUALIFIED\NAMESPACE\TO\MyClassDependencyBClass
 */
class MyClassTest extends \PHPUnit_Framework_TestCase
```

### Optional config
  
  * `bootstrap="vendor/autoload.php"` : Autoload file
  * `colors="true"` : Pretty output
  * `processIsolation="true"` : For [test isolation - different process](../Tests.md#rules-tests-isolation-different-process), but it could create edge cases
