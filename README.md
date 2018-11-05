# SetonoSyliusTrustpilotPlugin

[![Latest Version on Packagist][ico-version]][link-packagist]
[![Software License][ico-license]](LICENSE)
[![Build Status][ico-travis]][link-travis]
[![Quality Score][ico-code-quality]][link-code-quality]

Plugin for Sylius 1.3 to interate https://trustpilot.com to Sylius.

It sends an follow up emails to users who placed orders to leave 
feedback at Trustpilot.

## Installation

* Add plugin to `composer.json`:

    ```bash
    composer require setono/sylius-trustpilot-plugin
    ```

* Require bundle at application's `config/bundles.php`:

    ```php
    <?php
    // config/bundles.php
    
    return [
        // ...
        Setono\SyliusTrustpilotPlugin\SetonoSyliusTrustpilotPlugin::class => ['all' => true],
    ];
    ```

* Add plugin routing to application:

    ```yaml
    # config/routes.yaml
    setono_sylius_trustpilot_admin:
        resource: "@SetonoSyliusTrustpilotPlugin/Resources/config/admin_routing.yml"
        prefix: /admin
    ```

* Override `Customer` and `Order` entities:

    ```php
    <?php
    # src/AppBundle/Model/Customer.php
    
    declare(strict_types=1);
    
    namespace AppBundle\Model;
    
    use Setono\SyliusTrustpilotPlugin\Model\CustomerInterface;
    use Setono\SyliusTrustpilotPlugin\Model\CustomerTrait;
    use Sylius\Component\Core\Model\Customer as BaseCustomer;
    
    class Customer extends BaseCustomer implements CustomerInterface
    {
        use CustomerTrait;
    }
    ```
    
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!-- src/AppBundle/Resources/config/doctrine/model/Customer.orm.yml -->
    <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
                      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                      xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping
                                          http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd">
    
        <mapped-superclass name="AppBundle\Model\Customer" table="sylius_customer">
            <field name="trustpilotEnabled" column="trustpilot_enabled" type="boolean">
                <options>
                    <option name="default">1</option>
                </options>
            </field>
        </mapped-superclass>
    
    </doctrine-mapping>
    ```
    
    ```php
    <?php
    # src/AppBundle/Model/Order.php
    
    declare(strict_types=1);
    
    namespace AppBundle\Model;
    
    use Setono\SyliusTrustpilotPlugin\Model\OrderInterface;
    use Setono\SyliusTrustpilotPlugin\Model\OrderTrait;
    use Sylius\Component\Core\Model\Order as BaseOrder;
    
    class Order extends BaseOrder implements OrderInterface
    {
        use OrderTrait;
    }

    ```
    
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!-- src/AppBundle/Resources/config/doctrine/model/Order.orm.yml -->
    <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
                      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                      xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping
                                          http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd">
    
        <mapped-superclass name="AppBundle\Model\Order" table="sylius_order">
            <field name="trustpilotEmailsSent" column="trustpilot_emails_sent" type="smallint">
                <options>
                    <option name="default">0</option>
                </options>
            </field>
        </mapped-superclass>
    
    </doctrine-mapping>
    ```
    
    ```php
    <?php
    # src/AppBundle.php
  
    declare(strict_types=1);
    
    namespace AppBundle;
    
    use Doctrine\Bundle\DoctrineBundle\DependencyInjection\Compiler\DoctrineOrmMappingsPass;
    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\HttpKernel\Bundle\Bundle;
    
    final class AppBundle extends Bundle
    {
        public function build(ContainerBuilder $container)
        {
            parent::build($container);
    
            $container->addCompilerPass(DoctrineOrmMappingsPass::createXmlMappingDriver(
                [
                    realpath(__DIR__ . '/Resources/config/doctrine/model') => 'AppBundle\Model',
                ],
                ['doctrine.orm.entity_manager']
            ));
        }
    }
    ```
    
* Configure plugin:

    ```yaml
    # config/packages/_sylius.yml
    imports:  
        - { resource: "@SetonoSyliusTrustpilotPlugin/Resources/config/app/_sylius.yaml" }
    
    sylius_customer:
        resources:
            customer:
                classes:
                    model: AppBundle\Model\Customer
                  
    sylius_order:
        resources:
            order:
                classes:
                    model: AppBundle\Model\Order

    setono_sylius_trustpilot:
        trustpilot_email: "%env(APP_TRUSTPILOT_EMAIL)%"
    ```

* Put environment variable to `.env.dist`, `.env.*.dist` files:

    ```bash
    ###> setono/sylius-trustpilot-plugin ###
    APP_TRUSTPILOT_EMAIL=EDITME
    ###< setono/sylius-trustpilot-plugin ###
    ```

* Update your schema (for existing project)

```bash
# Generate and edit migration
bin/console doctrine:migrations:diff

# Then apply migration
bin/console doctrine:migrations:migrate
```

## Production notes

Make sure you configured next command to be executed on daily basis (with CRON):

```bash
bin/console setono:sylius-trustpilot:process --env=prod
```

Keep in mind this command probably should be executed in most
active time of day for your customers, e.g. not at 3:00.

# Extending

## Add custom `OrderEligibilityChecker` 

*   Write your custom class implementing `OrderEligibilityCheckerInterface`:
    
    Lets assume we don't want feedback for order less than $100. 

    ```php
    class CustomOrderEligibilityChecker implements OrderEligibilityCheckerInterface
    {
        public function isEligible(OrderInterface $order): bool
        {
            return $order->getTotal() >= 10000;
        }
    }
    ```
    
*   Tag it with `setono_sylius_trustpilot.order_eligibility_checker` tag
 
    ```xml
        <service id="AppBundle\Trustpilot\Order\EligibilityChecker\CustomOrderEligibilityChecker">
            <tag name="setono_sylius_trustpilot.order_eligibility_checker" />
        </service>
    ```

# Contribution

## Prepare to run test app

    ```bash
    cp tests/Application/.env.dist tests/Application/.env
    
    # Put actual values to environment variables
    nano tests/Application/.env
    
    set -a && source tests/Application/.env && set +a
    # OR (from tests/Application directory)
    # set -a && source .env && set +a
    ```

# (Manually) Test plugin

- Run application:
  (by default application have default config at `dev` environment
  and example config from `Configure plugin` step at `prod` environment)
  
    ```bash
    SYMFONY_ENV=dev
    cd tests/Application && \
        yarn install && \
        yarn run gulp && \
        bin/console doctrine:database:drop --force -e $SYMFONY_ENV && \
        bin/console doctrine:database:create -e $SYMFONY_ENV && \
        bin/console doctrine:schema:create -e $SYMFONY_ENV && \
        bin/console sylius:fixtures:load -e $SYMFONY_ENV && \
        bin/console server:run -d public -e $SYMFONY_ENV
    ```

- Log in at `http://localhost:8000/admin`
  with Sylius demo credentials:
  
  ```
  Login: sylius@example.com
  Password: sylius 
  ```

- ...

## Running plugin tests

  - PHPUnit

    ```bash
    $ vendor/bin/phpunit
    ```

  - PHPSpec

    ```bash
    $ vendor/bin/phpspec run
    ```

  - Behat (non-JS scenarios)

    ```bash
    $ vendor/bin/behat --tags="~@javascript"
    ```

  - Behat (JS scenarios)
 
    1. Download [Chromedriver](https://sites.google.com/a/chromium.org/chromedriver/)
    
    2. Download [Selenium Standalone Server](https://www.seleniumhq.org/download/).
    
    2. Run Selenium server with previously downloaded Chromedriver:
    
        ```bash
        $ java -Dwebdriver.chrome.driver=chromedriver -jar selenium-server-standalone.jar
        ```
        
    3. Run test application's webserver on `localhost:8080`:
    
        ```bash
        $ (cd tests/Application && bin/console server:run localhost:8080 -d public -e test)
        ```
    
    4. Run Behat:
    
        ```bash
        $ vendor/bin/behat --tags="@javascript"
        ```

### Opening Sylius with your plugin

- Using `test` environment:

    ```bash
    $ (cd tests/Application && bin/console sylius:fixtures:load -e test)
    $ (cd tests/Application && bin/console server:run -d public -e test)
    ```
    
- Using `dev` environment:

    ```bash
    $ (cd tests/Application && bin/console sylius:fixtures:load -e dev)
    $ (cd tests/Application && bin/console server:run -d public -e dev)
    ```

# TODO

- Admin with disable button
- Docs about process_latest_days
- Tests