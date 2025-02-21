# 1. PyTest Notes

- [1. PyTest Notes](#1-pytest-notes)
  - [1.1. Getting started with pytest](#11-getting-started-with-pytest)
  - [1.2. Writing Test function](#12-writing-test-function)
  - [1.3. Pytest Fixtures](#13-pytest-fixtures)
  - [1.4. Built-in fixtures](#14-built-in-fixtures)
  - [1.5. Parameterization](#15-parameterization)
    - [1.5.1. Function parameterization](#151-function-parameterization)
    - [1.5.2. Fixture parameterization](#152-fixture-parameterization)
  - [1.6. Test Strategy](#16-test-strategy)
  - [1.7. Configuration Files](#17-configuration-files)
  - [1.8. Coverage](#18-coverage)
  - [1.9. Mocking](#19-mocking)
  - [1.10. Tox and CI](#110-tox-and-ci)
  - [1.11. Debugging](#111-debugging)
    - [1.11.1. Flags for selecting which tests to run, in which order, and when to stop:](#1111-flags-for-selecting-which-tests-to-run-in-which-order-and-when-to-stop)
    - [1.11.2. Flags to control pytest output:](#1112-flags-to-control-pytest-output)
    - [1.11.3. Flags to start a command-line debugger:](#1113-flags-to-start-a-command-line-debugger)
  - [1.12. Third-party Plugins](#112-third-party-plugins)
  - [1.13. Building plugins](#113-building-plugins)
  - [1.14. Advanced Parameterization](#114-advanced-parameterization)

## 1.1. Getting started with pytest
* pytest identifies 
    * files named *_test.py or test_*.py
    * Functions named test_*()
    * Classes named TestSomething
* pytest outcomes-
    * PASSED (.)
    * FAILED (F) - exception in test
    * SKIPPED (s) - marked to skip
    * XFAIL (x) - marked as expected to fail
    * XPASS (X)  - expected to fail but passed
    * ERROR (E) - exception in fixture
* Convert dict to class object using ClassName(**object)
* Dunder keywords - Magic functions starting with double underscore
* `-r` flag with pytest shows the extra short test summary info. `-r` needs to be combined with a character like `s` (show why something skipped) or `a` (show all except pass) or `A` (all).
  
## 1.2. Writing Test function
* 2 ways to prove assertion-
    * `Assert` keyword
    * `pytest.fail(message)`
* Assertion helper function - Add `__tracebackhide__ = True` inside the test function if you don’t want the traceback of that function to be printed in the output if it fails
* To check if code is appropriately raising an exception as expected, you can test it as follows-
```python
def test_raises_with_info_alt():
    with pytest.raises(TypeError) as exc_info:
        cards.CardsDB()
    expected = "missing 1 required positional argument"
    assert expected in str(exc_info.value)
```
* Arrange Test Functions-
    * Arrange/Act/Assert or 
    * Given/When/Then
* If you have tests inside a class then call it as follows `test_file.py::TestClass::test_function`
* The `-k` parameter for pytest can be used to run only certain tests with some keyword in them. You can give expressions as well. For eg: “equality and diff” will run only those tests with both equality and diff keywords in their file name/class name/function name.

## 1.3. Pytest Fixtures
* Fixtures can have the setup and teardown code. If there is teardown, use `yield` keyword, if not then use return. Yield will return the content temporarily and once the code is done it comes back to the fixture and runs teardown.
* Use the `—setup-show` flag with pytest to see the order in which the code is run (setup, test name, teardown etc)
* Pytest shows output/print statements of a test function only if they fail. If you want to see it even if it passes use the `-s` flag.
* Different types of scopes-
    * **Function (default)** - once per test function
    * **Module** - once per test file
    * **Package** - once per test directory
    * **Session** - once per test session (invocation of pytest)
    * **Class** - once per test class
* Conftest doesn’t have to necessarily be in the same folder as the test file. It can be one/multiple directories up. The way to find the location of fixtures is by adding `—fixture` to pytest to see all built-in and custom made fixtures.
* `—fixtures-per-test` : gives the fixtures used by each test and where they are defined
* Use multiple fixture levels, the lower level fixtures having a broader scope and the higher level fixtures having smaller scope, in order to structure it more neatly.
* The scope of a fixture can be dynamically changed using the concept **PYTEST HOOKS**. For eg -
```python
def pytest_addoption(parser):
    parser.addoption(
        "--func-db",
        action="store_true",
        default=False,
        help="new db for each test",
    )

def db_scope(fixture_name, config):
    if config.getoption("--func-db", None):
        return "function"
    return "session"

@pytest.fixture(scope=db_scope)
def db():
    """CardsDB object connected to a temporary database"""
    with TemporaryDirectory() as db_dir:
        db_path = Path(db_dir)
        db_ = cards.CardsDB(db_path)
        yield db_
        db_.close()
```
if you pass `—func-db` flag while running pytest, it will set db fixture to function level, else set it to session level.
* **Autouse fixtures** - If a fixture is not used by a function, pytest won’t run the fixture code. If you still want the fixture to run anyway, that can be done by setting `autouse=True` inside `@pytest.fixture()` decorator. By default, it is function scope, that can be changed.
```python
@pytest.fixture(autouse=True, scope="session")
def footer_session_scope():
    """Report the time at the end of a session."""
    yield
    now = time.time()
    print("--")
    print("finished : {}".format(
        time.strftime("%d %b %X", time.localtime(now))))
    print("-----------------")


@pytest.fixture(autouse=True)
def footer_function_scope():
    """Report test durations after each function."""
    start = time.time()
    yield
    stop = time.time()
    delta = stop - start
    print("\ntest duration : {:0.3} seconds".format(delta))
```
 
* Fixtures can be renamed with `name=“name”` inside `pytest.fixture` decorator.

## 1.4. Built-in fixtures
* Useful Pytest in-built fixtures -
    * tmp_path
    * tmp_path_factory (session scope)
    * Capsys (Reads output and error, also can be used to disable output capture)
    * monkeypatch (Use it to patch functions and return a desired value for testing purposes only, kinda like mocking)
* CliRunner is another alternative to subprocess.run. Example code-
```python
import cards
from typer.testing import CliRunner

def test_version_v3():
    runner = CliRunner()
    result = runner.invoke(cards.app, ["version"])
    output = result.output.rstrip()
    assert output == cards.__version__
```
*  Monkeypatch can be used to patch different things differently. For example -
```python
def get_path():
    db_path_env = os.getenv("CARDS_DB_DIR", "")
    if db_path_env:
        db_path = pathlib.Path(db_path_env)
    else:
        db_path = pathlib.Path.home() / "cards_db"
    return db_path
```
* This function can be patched in 3 different ways -
    * Patch `env` variable [ monkeypatch.setenv("CARDS_DB_DIR", str(tmp_path)) ]
    * Patch `get_path` function itself [ monkeypatch.setattr(cards.cli, "get_path", tmp_path) ]
    * Patch `pathlib.path.home()` [ monkeypatch.setattr(cards.cli.pathlib.Path, "home", fake_home) ]
  
## 1.5. Parameterization
* Suppose you have to write tests that do the same thing but just have different inputs. While you can achieve that using a for loop and calling the same function with different inputs, it doesn’t tell you much about what fails if it fails. Instead you can use **parameterisation**. There are 3 ways to do parameterization:
  ### 1.5.1. Function parameterization
  For example -
    ```python
    @pytest.mark.parametrize(
        "start_summary, start_state",
        [
            ("write pytest book", "done"),
            ("create video course", "in prog"),
            ("write tdd book", "todo"),
        ],
    )
    def test_finish(cards_db, start_summary, start_state):
        initial_card = Card(summary=start_summary, state=start_state)
        index = cards_db.add_card(initial_card)

        cards_db.finish(index)

        card = cards_db.get_card(index)
        assert card.state == "done"
    ```
    * NOTE: Doesn't matter if you use list of tuples inside parameterize.
  ### 1.5.2. Fixture parameterization
  ```python
    @pytest.fixture(params=["done", "in prog", "todo"])
    def start_state(request):
        return request.param


    def test_finish(cards_db, start_state):
        c = Card("write tdd book", state=start_state)
        index = cards_db.add_card(c)
        cards_db.finish(index)
        card = cards_db.get_card(index)
        assert card.state == "done"
    ```
  ### 1.5.3. Using pytest hook (pytest_generate_tests)
  ```python
    def pytest_generate_tests(metafunc):
        if "start_state" in metafunc.fixturenames:
            metafunc.parametrize("start_state", ["done", "in prog", "todo"])


    def test_finish(cards_db, start_state):
        c = Card("write tdd book", state=start_state)
        index = cards_db.add_card(c)
        cards_db.finish(index)
        card = cards_db.get_card(index)
        assert card.state == "done"
    ```
* While running pytest, it calls it as 3 different tests with 3 different inputs.

## Markers

* You can use markers conditionally too. For example, if you want to skip a test if version of app is < 2:
```python
@pytest.mark.skipif(
    parse(cards.__version__).major < 2,
    reason="Card < comparison not supported in 1.x",
)
```
* **pytest.mark.xfail**:
  * When to use? Suppose you have a test that you know is going to fail (maybe due to a bug in some other library), you mark it as xfail. The output will be: 
    * xfail if the test actually failed 
    * xpass if the test that was supposed to fail ended up passing
  * You can add `strict=True` in the xfail function to indicate that it should fail if it actually passes.
    * Use Case? You would like to be notified that the test actually passed when it should have failed. This is useful when a lot of tests are automated today through CI jobs.
  * you can add the following in `pytest.ini` if you want this be the case for all the xfail tests:
```
[pytest]
xfail_strict = True
```
* `pytest --runxfail` is useful when i want to ensure tests dont have the xfail mark when running through CLI
*  You can create your own custom markers as well as follows:
   *  Add the following to `pytest.ini`:
    ```
    [pytest]
    markers =
        smoke: subset of tests # this is an example marker
        exception: check for expected exceptions # another example
    ```
   * Use the markers: `@pytest.mark.smoke` or `@pytest.mark.exception` above tests.
   * Call them while running pytest: `pytest -m <name of marker>`
   * Use Case? Helpful in organizing similar tests together
 * If you want every test in a file to have a mark you can do this: `pytestmark = [pytest.mark.smoke, pytest.mark.finish]`
 * You can apply pytest.mark on entire classes too
 * You can also mark pytest parameters like this:
    ```python
    @pytest.mark.parametrize(
        "start_state",
        (
            "todo",
            pytest.param("in prog", marks=pytest.mark.smoke),
            "done",
        ),
    )   
    ```
 * If you want misspelled markers to be identified during collection time itself and not runtime, run `pytest --strict-markers`. Fails a lot faster.
 *  To list all markers available: `pytest --markers` 
 *  You can combine markers with fixtures such that the markers provide inputs to the fixtures. for eg:
    *  Suppose you want to call the `cards_db` fixture with a marker above the test like `@pytest.mark.num_cards(n)` where `n` is the number of cards you would like to the fixture to have.
    *  You can do it as follows:
        ```python
        @pytest.fixture(scope="function")
        def cards_db(db, request, faker):
            # start empty
            db.delete_all()

            # support for `@pytest.mark.num_cards(<some number>)`

            # random seed
            faker.seed_instance(101)
            m = request.node.get_closest_marker("num_cards")
            if m and len(m.args) > 0:
                num_cards = m.args[0]
                for _ in range(num_cards):
                    db.add_card(
                        Card(summary=faker.sentence(), owner=faker.first_name())
                    )
            return db
        ```
    * NOTE: faker is a package and a pytest fixture that allows you to generate random stuff.

## 1.6. Test Strategy

* Determining test scope-
  * User visible functionality / features
  * Security
  * Performance
  * Loading
  * Input Validation
* While writing test cases, consider-
  * Happy path test case
  * Interesting sets of input
  * Diff starting states
  * Diff ending states
  * All possible error states


## 1.7. Configuration Files

* Determining Root dir - From whichever dir you run pytest, if `pytest.ini` is not found, it will keep going one dir up until it finds the file and that dir will be the root dir.
* Alternatives to `pytest.ini` -
  * tox.ini
  * pyproject.toml
  * setup.cfg (Least recommended - uses different parser)
* Having the `__init__.py` file in different subdirectories of the test folder having same test file names allows duplication of test file names.

## 1.8. Coverage

* Coverage can be calculated 2 ways-
  * Using the coverage package
  * Or the `cov` plugin in pytest
* Using cov plugin-
  * `pip install pytest-cov`
  * `pytest <directory_of_tests> --cov=<dir_to_source_code> --cov-report=term-missing`
*  Directly using coverage command-
   *  `coverage run --source-<dir_to_source_code> -m pytest <directory_of_tests>`
   *  `coverage report --show-missing`
*  `.coveragerc` is the configuration file for coverage.
*  Example contents in the file:
    ```
    [paths]
    source =
        cards_proj/src/cards
        */site-packages/cards
    ```
* To generate a HTML report for missed lines in the coverage-
  * `--cov-report=html`
  * or `coverage run` followed by `coverage html`
* If you dont want a piece of code to be excluded from coverage, add `#pragma: no cover` next to it. Usually used next to `__name__ == __main__`
* Another way to exclude something from coverage, add the following to `.coveragerc`-
  ```
  [report]
  exclude_also =
    if __name__ == .__main__.:
  ```
*  You can cover tests also by adding `--cov=test_dir`. If you copy test function and forget to change test name, this is a good way to catch it.
*  If you want to set min coverage level run `pytest --cov=cards --cov-fail-under=80`
*  If you have the test functions and the src code in the same file, you don't need to import pytest. Only import if you want to use the pytest wrappers etc. You can import it as follows-
  ```python
  # src code
  def main():
    pass
  if __name__ == '__main__':
    main()
  else
    import pytest
  # test functions
  ```

## 1.9. Mocking

* You can test CLI Commands with `CLIRunner` in `pytest.testing`.
* Example-
    ```python
    from typer.testing import CLiRunner

    runner = CLiRunner()

    def test_typer_runner():
        result = runner.invoke(app, ["list", "-o", "brian"])
        print(f"list: {result.stdout}")
    ```
    where app is 
    ```python
    app = typer.Typer(add_completion=False)
    ```
* `shlex.split()` can be used to split CLI commands
* Mocking can be done as follows-
  ```python
  from unittest import mock #mock is part of unittest module now
  def test_mock_version():
    with mock.patch.object(cards, "__version__", "1.2.3"):
        result = cards_cli("version")
  ```
* Suppose you want to mock a class- `db = cards.CardsDB(db_path)`
  ```python
  with mock.patch.object(cards, "CardsDB") as MockCardsDB:
    MockCardsDB.return_value.path.return_value = "/foo/"
  ```
* If interface of an API changes, the mock objects dont throw an error. In order to prevent the mock object from inventing interfaces, use `autospec=True` in the `mock.path.object`.
* Ensuring transition of states happen correctly-
  ```python
  def test_count(mock_cardsdb):
    cards_cli("start 23")
    mock_cardsdb.start.assert_called_with(23)
  ```
* Adding side effect for exceptions - `mock_cardsdb.start.side_effect = cards.api.InvalidCardId`
* Listing cards example-
  ```python
    expected_output = """\
                                    
    ID   state   owner   summary    
    ──────────────────────────────── 
    1    todo            some task  
    2    todo            another
                                    
    """


    def test_list(mock_cardsdb):
        some_cards = [
            cards.Card("some task", id=1),
            cards.Card("another", id=2),
        ]
        mock_cardsdb.list_cards.return_value = some_cards
        output = cards_cli("list")
        assert output.strip() == expected_output.strip()
  ```
* **REMEMBER**: Mocks dont test behaviour, just the implementation
* plugins for mocking-
  * Databases: pytest-postgresql, pytest-mongo etc
  * HTTP servers: pytest-httpserver
  * Requests: responses, betamax
  * Misc: pytest-rabbitmq, pytest-redis

## 1.10. Tox and CI

* CI - combining integrations from multiple people
* tox 
  * mini local CI system
  * build project, create venv
* Simple tox file syntax-
  ```
  [tox]
  envlist = py312

  [testenv]
  deps = pytest
  commands = pytest
  ```
* tox envs can be run parallely as well. For example -
  ```
  [tox]
  envlist = py38, py39, py312
  
  [testenv]
  deps = pytest
  commands = pytest
  ```
  with the `-p` parameter : `tox -c tox.ini -p`
* To pass pytest parameters through tox, add `posargs` : `pytest --cov=cards {posargs}` and while running tox command add the `--` seperator - `tox -c tox.ini -e py312 -- -v -k test_version`
* If you want to add src code path to syspath, add the following to tox.ini-
  ```
  [pytest]
  pythonpath = src
  ```
## 1.11. Debugging

### 1.11.1. Flags for selecting which tests to run, in which order, and when to stop:
 
* `--lf` / `--last-failed`: Runs just the tests that failed last
* `--ff` / `--failed-first`: Runs all the tests, starting with the last failed
* `-x` / `--exitfirst`: Stops the tests session after the first failure
* `--maxfail=num`: Stops the tests after num failures
* `--nf` / `--new-first`: Runs all the tests, ordered by file modification time
* `--sw` / `--stepwise`: Stops the tests at the first failure. Starts the tests at the last failure next time
* `--sw-skip` / `--stepwise-skip`: Same as --sw, but skips the first failure

### 1.11.2. Flags to control pytest output:
* `-v` / `--verbose`: Displays all the test names, passing or failing
* `--tb=[auto/long/short/line/native/no]`: Controls the traceback style
* `-l` / `--showlocals`: Displays local variables alongside the stacktrace

### 1.11.3. Flags to start a command-line debugger:
* `--pdb`: Starts an interactive debugging session at the point of failure
* `--trace`: Starts the pdb source-code debugger immediately when running each test

## 1.12. Third-party Plugins

* pytest-order - specifify order of tests
* pytest-randomly - Randomize test
* pytest-repeat
* pytest-rerunfailures 
* pytest-xdist - running tests in parallel
* pytest-instafail
* pytest-sugar - make it pretty
* pytest-html - generate html output for tests
* pytest-benchmark - how long it takes to run the tests
* pytest-freezegun - Freeze the time for any app that runs code to retrieve time

## 1.13. Building plugins

* Three main hook functions can be used:
  * `pytest_configure(config)` - Configure markers
  * `pytest_addoption(parser)` - Add pytest flag
  * `pytest_collect_modifyitems(config, items)` - This is called after test collection happens and you know what to do and want to modify the behaviour of pytest
  * To create a package out of the plugin, do the following:
    * `pip install flit` (Flit is used to put python packages and modules on PyPI)
    * `flit init` - It creates a pyproject file and license file.
    * `flit build` (add `--no-use-vcs` is you dont want it to use version control)
  * [packaging.python.org](https://packaging.python.org/en/latest/) has more info on python packaging. Info on pyproject.toml.
  * If you want to install packages in editable mode, run `pip install -e`
  * If you want to test your plugins, use `pytester`. It is not turned on by default, you need to turn it on by adding `pytest_plugins = ["pytester"]` in `conftest.py`.
  * Examples of how to use pytester to run tests:
    ```python
    @pytest.fixture()
    def examples(pytester):
        pytester.copy_example("examples/test_slow.py")


    def test_skip_slow(pytester, examples):
        result = pytester.runpytest("-v")
        result.stdout.fnmatch_lines(
            [
                "*test_normal PASSED*",
                "*test_slow SKIPPED (need --slow option to run)*",
            ]
        )
        result.assert_outcomes(passed=1, skipped=1)


    def test_run_slow(pytester, examples):
        result = pytester.runpytest("--slow")
        result.assert_outcomes(passed=2)
    ```
  
## 1.14. Advanced Parameterization

*  If you have parameterization for your tests where you pass complex classes / objects, pytest wont show what parameter is passed exactly. It shows up as obj1, obj2 etc.
*  To overcome this, use `id=` while calling the parameterize marker.
    *  `id=str` pass object as string in pytest output.
    *  `id=custom_function` function that returns object property or anything.
    *  `id=list` where list is a custom list of names.
*  You can also parameterize with dynamic values (generators) as follows:
    ```python
    def text_variants():
      variants = {
        "short": "x",
        "with spaces": "x y z",
        "end in spaces": "x   ",
      }
      for key,value in variants.items():
        yield pytest.param(value, id=key)
    ```
* Parameterization with multiple values can be done in 2 different ways:
  * ```python
      @pytest.mark.parametrize(
        "summary, owner, state",
        [
            ("short", "First", "todo"),
            ("short", "First", "in prog"),
            # ...
        ],
      )
      def test_add_lots(cards_db, summary, owner, state)
      ```
  * ```python
      summaries = ["short", "a bit longer"]
      owners = ["First", "First M. Last"]
      states = ["todo", "in prog", "done"]


      @pytest.mark.parametrize("state", states)
      @pytest.mark.parametrize("owner", owners)
      @pytest.mark.parametrize("summary", summaries)
      def test_stacking(cards_db, summary, owner, state)
      ```
* Indirect fixtures can be used by passing fixture name and list of parameters. For eg-
  ```python
    @pytest.mark.parameterize("user", ["admin", "team_member", "visitor"], indirect=["user"])
    def test_user(user):
  ```
  This will run the fixture named user first and then pass the parameters corrresponding to that fixture.
* If a fixture is parameterized, it is also possible to select a subset of the parameterized fixture.
* 