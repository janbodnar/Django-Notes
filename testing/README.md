# Testing in Django



The key components of testing in Django:

1. **TestCase**: A subclass of Python’s standard `unittest.TestCase` tailored for Django applications.
   It provides tools for testing models, views, forms, and more.  
3. **Test Client**: Simulates requests to your Django views, allowing you to test response statuses,
   content, and behavior.  
5. **Fixtures**: Files containing serialized data used to populate the test database with initial data
    before tests run.  
7. **Assertions**: Methods to verify that the code behaves as expected, such as `assertEqual`, `assertTrue`, and `assertRaises`.
8. **Isolation**: Each test runs in a separate environment to ensure tests do not interfere with each other, using
   a temporary test database.  



## The tests directory 

We can move `tests.py` file into the `test` directory, where we create component tests such as  
`test_models.py`. Also create the `__init__.py` file.  We run tests then with `py manage.py test`.  


### Django's TestCase

Django's `TestCase` is a subclass of Python’s standard `unittest.TestCase`. It is designed specifically for 
testing Django applications. `TestCase` provides a set of tools and methods for creating and running tests in  
Django projects, making it easier to test your models, views, forms, and other components.

### Basic Characteristics of Django's TestCase

1. **Database Setup**:
   - `TestCase` creates a separate test database for running tests.
   - The test database is a temporary database that is created and destroyed automatically.
   - Migrations are applied to the test database, ensuring it has the same schema as your development database.

2. **Fixtures**:
   - You can use fixtures to load initial data into the test database.
   - Fixtures are files containing serialized data that can be used to populate the database before tests run.

3. **setUp Method**:
   - The `setUp` method is called before each test method.
   - It is used to set up any preconditions needed for the tests.
   - You can create objects or set up other test-specific data here.

4. **tearDown Method**:
   - The `tearDown` method is called after each test method.
   - It is used to clean up any resources or data after the tests run.
   - Typically, you won't need to override this method in Django tests, as the test runner handles cleanup.

5. **Assertions**:
   - `TestCase` provides various assertion methods to check the outcomes of your tests.
   - Common assertions include `assertEqual`, `assertTrue`, `assertFalse`, `assertRaises`, and more.
   - These assertions help you verify that your code behaves as expected.

6. **Client**:
   - `TestCase` includes a test client that allows you to simulate requests to your Django views.
   - The test client can be used to send GET, POST, and other HTTP requests to your views.
   - This helps you test your views' responses and behavior.

7. **Isolation**:
   - Each test runs in isolation, ensuring that the test environment is clean and consistent.
   - The state of the database and other resources is reset between tests.

### Example

Here's a simple example to illustrate the use of Django's `TestCase`:

```python
from django.test import TestCase
from .models import Product

class ProductTestCase(TestCase):
    
    def setUp(self):
        self.product = Product.objects.create(
            name="Test Product",
            price=10.00,
            stock=100
        )
    
    def test_product_creation(self):
        self.assertIsNotNone(self.product.id)
        self.assertEqual(self.product.name, "Test Product")
        self.assertEqual(self.product.price, 10.00)
        self.assertEqual(self.product.stock, 100)
    
    def test_product_str(self):
        self.assertEqual(str(self.product), "Test Product")
```

In this example:

- The `setUp` method creates a `Product` instance before each test method.
- The `test_product_creation` method checks if the product is created correctly.
- The `test_product_str` method checks if the string representation of the product is as expected.

Using Django's `TestCase` class, you can ensure that your tests are run in a controlled and isolated  
environment, making it easier to identify and fix issues in your Django application.  

If you have any more questions or need further assistance, feel free to ask!


## Common test commands 

Common Django test commands:

| Command | Description |
|---------|-------------|
| `python manage.py test` | Run all tests. |
| `python manage.py test -v 2` | Run tests with detailed output. |
| `python manage.py test myapp` | Run tests for a specific app. |
| `python.manage.py test myapp.tests.TestCaseName` | Run tests for a specific test case. |
| `python.manage.py test myapp.tests.TestCaseName.test_method_name` | Run a specific test method. |
| `python manage.py test --reuse-db` | Run tests with database reuse. |
| `python manage.py test --parallel` | Run tests in parallel. |
| `DJANGO_SETTINGS_MODULE=myproject.settings.test python manage.py test` | Run tests with specified settings. |
| `python manage.py test --failfast` | Stop the test suite after the first failure. |
| `python manage.py test --keepdb` | Preserve the test database between test runs. |
| `python manage.py test --reverse` | Run tests in the reverse order. |
| `python manage.py test --debug-mode` | Run tests with debug mode enabled. |
| `python manage.py test --tag=special` | Run tests with a specific tag. |
| `python manage.py test --pdb` | Enter pdb on test failure. |
| `python manage.py test --debug-sql` | Show SQL queries as they're executed. |
| `python manage.py test --pattern="tests_*.py"` | Run tests matching a specific pattern. |
| `python manage.py test --timing` | Display the timing of each test. |
| `python manage.py test --verbosity=3` | Increase verbosity of output to maximum. |
| `python manage.py test --noinput` | Don't prompt for any input. |
| `python manage.py test --addrport=localhost:8081` | Run a test server on a specific address and port. |
| `python manage.py test --testrunner=myproject.custom_test_runner` | Use a custom test runner. |
| `python manage.py test --no-color` | Disable color output in test results. |
| `python manage.py test --parallel=N` | Run tests in parallel with N processes. |
| `python manage.py test --liveserver=localhost:8081` | Use a specific server address for live server tests. |
| `python manage.py test --reverse-sequential` | Run tests in reverse sequential order. |

