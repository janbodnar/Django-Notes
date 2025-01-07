# Testing in Django


## Common test commands 

Of course! Here are 25 common Django test commands presented in a table:

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

