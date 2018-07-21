+++
title = "Integration tests with Travis CI"
author = "Sander Knape"
date = 2016-09-20T19:46:02+02:00
draft = false
tags = ["automation", "integration testing", "php", "travis ci"]
categories = []
+++
Do you write integration tests? What about unit tests? I believe that more people say "Yes" to the second question than to the first. Which is kinda weird - for many applications, it really isn't that hard to write integration tests. It might not even be necessary to setup your own infrastructure to run these tests. Many CI tools these days allow you to install databases, queues and such on their build agents. With your external dependencies available on your build server, a complementary sets of tests can be run next to your unit tests.  

In this blog post I will show how easy it is to setup an integration test using Travis CI. Before doing so, let's dive into a bit of theory; what exactly is the difference between an integration test and an unit test?

# Integration tests vs. Unit tests

Let's begin with the perhaps more familiar unit test. With unit testing, you test an isolated, typically small piece of code such as a function or a class. You either mock or stub external dependencies. These dependencies can be other classes or functions within your code. They can also be completely different applications, API endpoints, databases, queues, and much more. Whenever you test a function that has such an external dependency, you inject predetermined behavior or state into its mock. Therefore you can test: what happens to my function when the API call returns response X? What if returns response Y?  

With integration testing you test the combination of different components. The definition of _component_ is key here: it can be a class, a service/application, or even a set of applications. You test how your components integrate with other components. The key difference with unit testing then is that with unit tests, you mock or stub external dependencies. With an integration test you do exactly the opposite - you actually _do_ connect with a MySQL database, an API endpoint, a RabbitMQ queue or a Memcache instance. If you connect with multiple external dependencies, you might still mock dependencies to test the integration with a specific one. The challenge here becomes to setup your external dependencies correctly. Instead of adding the correct state to a mock, you will put the correct state in a database. This is exactly what we will doing in this blog post.

# Introducing: _The Doubler_

I've set up a very simple application that integrates with an external database. What this application does is quite impressive: it multiplies numbers by two. The single-file application can be found in my [Github repository](https://github.com/SanderKnape/TravisIntegrationTest). If you want to follow along while reading this blogpost, I would suggest to create a (temporary) repository in Github and add the code as you follow along. In the end we will integrate the repository with Travis CI to execute the integration test.  

My repository also contains a [Vagrantfile](https://www.vagrantup.com/) that you can use to start the application. If you are not familiar with Vagrant: it spawns a Virtual Machine using Virtualbox (and has support for other virtualization technologies and Docker). In our case, it uses a CentOS7 base box and installs PHP7, MySQL5.6 and of course our application.  

_The doubler_ is a batch-job PHP application that integrates with MySQL. Running the application doubles every number found in a MySQL table and stores it in that same table. The schema of the table is as follows;

```sql
CREATE TABLE numbers(
  number INT(11) NOT NULL,
  number_calculated INT(11) DEFAULT NULL
);
```

Every value in the `number` field is multiplied by two and stored in the `number_calculated` field. The total contents of the PHP application is only a few lines:

```php
$mysql_host = getenv('MYSQL_HOST') ?: 'localhost';
$mysql_user = getenv('MYSQL_USER') ?: 'root';
$mysql_password = getenv('MYSQL_PASSWORD') ?: '';

$connection_string = "mysql:host={$mysql_host};dbname=numbers";
$db = new PDO($connection_string, $mysql_user, $mysql_password);

$db->exec("UPDATE numbers SET number_calculated = number*2 WHERE number_calculated IS NULL");
```

As we will see later: in Travis CI I overwrite the environment variables with the correct settings for configuring to the MySQL instance installed on the Travis CI build container. This makes it easy for me to locally develop the application with my local settings while the application will also work in the Travis environment.  

Running the application is done by executing the `doubler.php` file with the php-cli:

```bash
php -f /vagrant/doubler.php
```

# Creating the integration tests

Properly configuring the integration test requires us to setup three different steps;

1.  _Fill the database with fixtures_. A fixture is a set of test data. In our case: a set of numbers added to the `numbers` field in MySQL.
2.  _Run our application_. We will 'run' our application against the just filled database. In our very simple setup this means: executing the _doubler.php_ file.
3.  _Review if the correct contents are added to the database_. We now check if the `number_calculated` fields are filled with the correct numbers.

## Inserting test data

Before inserting the test data a database with a schema is required. A simple script sets this up:

```php
#!/usr/bin/env php
<?php

$mysql_host = getenv('MYSQL_HOST') ?: 'localhost';
$mysql_user = getenv('MYSQL_USER') ?: 'root';
$mysql_password = getenv('MYSQL_PASSWORD') ?: '';

$connection_string = "mysql:host={$mysql_host}";
$db = new PDO($connection_string, $mysql_user, $mysql_password);

$schema = file_get_contents(dirname(__FILE__) . '/schema.sql');
$db->exec($schema);
```

The first line in the script is what is called a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)). As we will be executing this script from the command line, the shell needs to know which interpreter to use to execute this script. The first line therefore tells it to look into the $PATH of the current environment and use the PHP executable.  

As we did before, we use environment variables to get the credentials for the database in case we are not running the script locally. For readability I separated the script that creates the MySQL schema into a different file. This file sets up the schema as shown in the previous section.  

Next we will insert the test data. Similar to writing unit tests it is important to think of edge cases that you want to test. In case of multiplying numbers, you might you want to test the correct execution of negative numbers, 0 (zero) and large numbers.

```php
#!/usr/bin/env php
<?php

$mysql_host = getenv('MYSQL_HOST') ?: 'localhost';
$mysql_user = getenv('MYSQL_USER') ?: 'root';
$mysql_password = getenv('MYSQL_PASSWORD') ?: '';

$connection_string = "mysql:host={$mysql_host};dbname=numbers";
$db = new PDO($connection_string, $mysql_user, $mysql_password);

$count = $db->exec("
  INSERT INTO numbers
    (number)
  VALUES
    (1),
    (2),
    (10),
    (42),
    (0),
    (-100),
    (123456789),
    (-33);
");
```

## Running the application

Running the application should essentially be the easier part. With a batch job process like in this example, it's especially easy as its core job is to be executed, process some data and exit. With a web application including a frontend UI, this part will be harder. Keep in mind that you want to test the integration with an external dependency: does my database contain the correct data after I add an item to the shopping cart? Does it disappear when removing it from my shopping cart?  

Back to the doubler: running this application is as easy as executing the _doubler.php _file. It will find the test data in the database and perform the multiplications.

```php
require dirname(__FILE__) . '/../doubler.php';
```

Next, it is time to find out if the correct data is inserted into the table.

## Assert the results

Querying the MySQL database will show us if the correct data is inserted by the doubler. After fetching all rows from the table, I use a simple helper function to assert whether the expected values are present in every row. It is important to return the proper non-zero exit code when an error is found. This is a way for the executor of the script to know if the script was executed successfully or not. With 0 (zero), everything is fine. A value > 0 means something went wrong.

```php
$stmt = $db->prepare("SELECT number_calculated FROM numbers");
$stmt->execute();

$result = $stmt->fetchAll();

testCalculatedNumber(0, 2, $result);
testCalculatedNumber(1, 4, $result);
testCalculatedNumber(2, 20, $result);
testCalculatedNumber(3, 84, $result);
testCalculatedNumber(4, 0, $result);
testCalculatedNumber(5, -200, $result);
testCalculatedNumber(6, 246913578, $result);
testCalculatedNumber(7, -66, $result);

echo "All numbers OK";

// a simple helper function to easily test for the correct data
function testCalculatedNumber($index, $expected, $result)
{
  $number_calculated = (int)$result[$index]['number_calculated'];

  if($number_calculated !== $expected) {
    echo "Expected number calculated to be '{$expected}', got '{$number_calculated}'";

    // exit with the correct error code so Travis CI picks this up as a failed test
    exit(1);
  }
}
```

Again, be sure to check out my [Github repository](https://github.com/SanderKnape/TravisIntegrationTest) for the full project including tests.

# Configuring the integration test on Travis CI

Finally, all previous scripts will come together in the Travis CI configuration. We add a `.travis.yml` file to the project that Travis CI uses to correctly build and test the application.

```yaml
language: php
php:
  - '7.0'

env:
  - MYSQL_HOST=127.0.0.1 MYSQL_USER=root

services:
  - mysql

before_script: ./tests/setup_mysql.php
script: ./tests/integration_test.php

```

The top values are to tell Travis CI which language and version we are running. Next, we set the environment variables to the [default MySQL credentials for Travis CI](https://docs.travis-ci.com/user/database-setup/#MySQL). In the services array, we tell Travis CI to install MySQL on the build container. Finally, we setup MySQL in the `before_script` step and set the `script` that Travis CI must execute for testing the application. That is all there is to it; be sure to check out the [Travis CI documentation](https://docs.travis-ci.com/) to learn more about the `.travis.yml` file.  

It is now time to link Travis CI to our Github repository for it to can fetch the contents and execute the integration tests. If you are following along: be sure that all files are in the correct place in your repository so that Travis CI can find them. Visit [travis-ci.org](https://travis-ci.org/) and click the button to sign in with Github. Give Travis CI access to your account. You can specify exactly which repositories Travis CI should be given access to. Enable your repository, push new changes, and Travis CI should start with the very first test of your repository! Travis CI will now test your repository after each push to the master branch. Hopefully all goes well and the build will turn green. If not, note the error that is returned and apply the appropriate fixes.  

But what if your build fails? We can introduce a "bug" to our application and see what happens when we push this code to our repository. Let's say we change the line that doubles all numbers in the MySQL table to this:

```php
$db->exec("UPDATE numbers SET number_calculated = number*3 WHERE number_calculated IS NULL");
```

U-oh; we are now multiplying each number with three which should certainly cause the integration test to fail. Push this change to your repository and see your commit fail (or see the "Introduce Bug" commit in [my commit history](https://github.com/SanderKnape/TravisIntegrationTest/commits/master); click on the red X to see more information):  

![a failing integration test because of a bug](/images/bug.png)  

Revert your change and see your build turn green again:  

![and the integration test succeeds again after fixing the bug](/images/nobug.png)  

That's it! We have Travis CI running an integration test on our application.

# Conclusion

In this blog post we created an integration test for a very simple batch job application. This is a relatively simple use case of course. What happens when your process contains multiple (batch) application and databases? What about testing a web application?  

These are some topics I hope to pick up in future blog posts. For now, I hope this post has shown the ease with which you can use a CI tool such as Travis CI for running integration tests.
