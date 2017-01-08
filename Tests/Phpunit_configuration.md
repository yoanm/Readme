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
  checkForUnintentionallyCoveredCode="true"
  bootstrap="vendor/autoload.php"
  colors="true"
>
  <listeners>
        <listener class="StrictModeListener" file="tests/StrictModeListener.php"/>
  </listeners>

  ...

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

### Rules validated with [configuration reference](#configuration-reference)
#### Mandatory

  * [Early stop](../Tests.md#rules-early-stop)

    * `stopOnError="true"`
    * `stopOnFailure="true"`

  * [Strict mode](../Tests.md#rules-strict-mode)

    * [Exit status](../Tests.md#exit-status) : PhpUnit command will return a failed status if a failed or on error test exist
    * [Fails if](../Tests.md#rules-strict-mode-fails-if)
      * [Php errors](../Tests.md#rules-strict-mode-fails-if-php-errors)

        * `convertErrorsToExceptions="true"`
        * `convertNoticesToExceptions="true"`
        * `convertWarningsToExceptions="true"`
    
      * [Risky tests](../Tests.md#rules-strict-mode-fails-if-risky-tests) (requires [`listener`](#listener))

        * `beStrictAboutOutputDuringTests="true"`
        * `beStrictAboutChangesToGlobalState="true"`
        * `beStrictAboutTestsThatDoNotTestAnything="true"`

  * [Risky test](../Tests.md#rules-risky-tests)

    * [No Output](../Tests.md#rules-risky-tests-output) with `beStrictAboutOutputDuringTests="true"` 
    * [No globals manipulation](../Tests.md#rules-risky-tests-manipulate-globals) with `beStrictAboutChangesToGlobalState="true"`
    * [No test that test nothing](../Tests.md#rules-risky-tests-test-nothing) with `beStrictAboutTestsThatDoNotTestAnything="true"`
  
  * [Test isolation](../Tests.md#rules-tests-isolation)
    
    * [Globals backup](../Tests.md#rules-tests-isolation-globals) with `backupGlobals="true"`
      
      *Required by `beStrictAboutChangesToGlobalState="true"`*

    * [Static class member backup](../Tests.md#rules-tests-isolation-static-class-member) with `backupStaticAttributes="false"`
  
  * [Real coverage](../Tests.md#rules-real-coverage)
    
    * [No coverage overflow](../Tests.md#rules-real-coverage-overflow)
      
      * `forceCoversAnnotation="true"`

        If no annotation, no coverage.

        Symply add the following as test class or test method comment

        ```
        /**
         * @covers FULLY\QUALIFIED\NAMESPACE\TO\MyClass
         */
        ```

    * [Risky tests does not count in coverage](../Tests.md#rules-real-coverage-risky-tests)
    
      * [No Output](../Tests.md#rules-risky-tests-output) with `beStrictAboutOutputDuringTests="true"` (requires [`listener`](#listener))
      * [No globals manipulation](../Tests.md#rules-risky-tests-manipulate-globals) with `beStrictAboutChangesToGlobalState="true"`
      * [No test that test nothing](../Tests.md#rules-risky-tests-test-nothing) with `beStrictAboutTestsThatDoNotTestAnything="true"`
    
  * <a name="listener"></a>\<listener>
      
      Listener will validate following mandatory rules

      * [Strict mode - fails if - risky tests](../Tests.md#rules-strict-mode-fails-if-risky-tests)

        * Required by 
      
          * `beStrictAboutOutputDuringTests="true"` ([No Output](../Tests.md#rules-risky-tests-output))
          * `beStrictAboutChangesToGlobalState="true"` ([No globals manipulation](../Tests.md#rules-risky-tests-manipulate-globals))
          * `beStrictAboutTestsThatDoNotTestAnything="true"` ([No test that test nothing](../Tests.md#rules-risky-tests-test-nothing))

      * [Real coverage - risky tests  does not count in coverage](../Tests.md#rules-real-coverage-risky-tests) for some specific kinds of risky test   
      
          * Required by `beStrictAboutOutputDuringTests="true"` ([No Output](../Tests.md#rules-risky-tests-output))

#### Optional
  * [Test doc - tested class](../Tests.md#rules-test-documentation-tested-class-description) : by using `@covers`
      
    *In fact, required if coverage is used as configuration uses `forceCoversAnnotation="true"`*

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
