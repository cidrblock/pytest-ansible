# pytest-ansible

[![Build Status](https://github.com/ansible-community/pytest-ansible/actions/workflows/tox.yml/badge.svg)](https://github.com/ansible-community/pytest-ansible/actions/workflows/tox.yml)
[![Version](https://img.shields.io/pypi/v/pytest-ansible.svg)](https://pypi.python.org/pypi/pytest-ansible/)
[![License](https://img.shields.io/pypi/l/pytest-ansible.svg)](https://pypi.python.org/pypi/pytest-ansible/)
[![Supported Python Versions](https://img.shields.io/pypi/pyversions/pytest-ansible.svg)](https://pypi.python.org/pypi/pytest-ansible/)

This repository now extends its support to two additional plugins `pytest-molecule` and `pytest-ansible-units`. These plugins enhance the functionality of the pytest framework by providing specialized features for Ansible testing workflows. The repository includes a `pytest` plugin that introduces fixtures for executing `ansible` modules and examining `ansible_facts`. Unlike using the `subprocess` module to call ansible and parsing stdout for outcomes, the `pytest-ansible` plugin offers improved reliability by returning JSON data from modules which you can inspect and act on, much like with an ansible [playbook](https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html).

The [pytest-molecule](https://github.com/ansible-community/pytest-molecule) plugin enables `pytest` discovery of all `molecule.yml` files inside the codebase and runs them as `pytest` tests. It enables you to include Molecule scenarios as part of your pytest test suite, allowing you to thoroughly test your Ansible roles and playbooks across different scenarios and environments. Once you install `pytest-ansible` you should be able to just run `pytest` in order to run `molecule` on all roles and scenarios.

The [pytest-ansible-units](https://github.com/ansible-community/pytest-ansible-units) plugin allows ansible collection's unit tests to be run with only `pytest`. It offers a focused approach to testing individual Ansible modules. With this plugin, you can write and execute unit tests specifically for Ansible modules, ensuring the accuracy and reliability of your module code. This is particularly useful for verifying the correctness of module behavior in isolation.

## Installation

Install this plugin using `pip`

```bash
pip install pytest-ansible
```

## Running molecule scenarios using pytest

Molecule scenarios can be tested using 2 different methods.

**Recommended:**

Add a `test_molecule.py` file to the `tests` directory of the ansible collection:

```
# Tests for molecule scnearios

def test_scenario(molecule_scenario):
    """Run molecule for each scenario"""
    proc = molecule_scenario.test()
    assert proc.returncode == 0
```

The `molecule_scenario` fixture provides parameterized molecule scenarios discovered in the collection's `extensions/molecule` directory.

`molecule test -s <scenario>` will be run for each scenario and a completed subprocess returned from the `test()` call.

**Legacy:**

Run molecule with the `--molecule` command line parameter to inject each molecule directory found in the current working directory. Each scenarion will be injected as an external test in the the tests available for pytest.

## Fixtures and helpers for use in tests

### Usage

Once installed, the following `pytest` command-line parameters are available:

```bash
pytest \
    [--inventory <path_to_inventory>] \
    [--extra-inventory <path_to_extra_inventory>] \
    [--host-pattern <host-pattern>] \
    [--connection <plugin>] \
    [--module-path <path_to_modules] \
    [--user <username>] \
    [--become] \
    [--become-user <username>] \
    [--become-method <method>] \
    [--ask-become-pass] \
    [--limit <limit>] \
    [--ansible-unit-inject-only] \
    [--molecule] \
    [--molecule-unavailable-driver] \
    [--skip-no-git-change] \
    [--check]
```

### Inventory

Using ansible first starts with defining your inventory. This can be done in
several ways, but to start, we'll use the `ansible_adhoc` fixture.

```python
def test_my_inventory(ansible_adhoc):
    hosts = ansible_adhoc()
```

In the example above, the `hosts` variable is an instance of the `HostManager`
class and describes your ansible inventory. For this to work, you'll need to
tell `ansible` where to find your inventory. Inventory can be anything
supported by ansible, which includes an [INI
file](http://docs.ansible.com/ansible/latest/intro_inventory.html) or an
executable script that returns [properly formatted
JSON](http://docs.ansible.com/ansible/latest/intro_dynamic_inventory.html).
For example,

```bash
pytest --inventory my_inventory.ini --host-pattern all
```

or

```bash
pytest --inventory path/to/my/script.py --host-pattern webservers
```

or

```bash
pytest --inventory one.example.com,two.example.com --host-pattern all
```

In the above examples, the inventory provided at runtime will be used in all
tests that use the `ansible_adhoc` fixture. A more realistic scenario may
involve using different inventory files (or host patterns) with different
tests. To accomplish this, the fixture `ansible_adhoc` allows you to customize
the inventory parameters. Read on for more detail on using the `ansible_adhoc`
fixture.

### Extra Inventory

Using ansible first starts with defining your extra inventory. This feature was added in version 2.3.0, and is intended
to allow the user to work with two different inventories. This can be done in several ways, but to start,
we'll use the `ansible_adhoc` fixture.

For example,

```bash
pytest --inventory my_inventory.ini --extra-inventory my_second_inventory.ini --host-pattern host_in_second_inventory
```

#### Fixture `ansible_adhoc`

The `ansible_adhoc` fixture returns a function used to initialize
a `HostManager` object. The `ansible_adhoc` fixture will default to parameters
supplied to the `pytest` command-line, but also allows one to provide keyword
arguments used to initialize the inventory.

The example below demonstrates basic usage with options supplied at run-time to
`pytest`.

```python
def test_all_the_pings(ansible_adhoc):
    ansible_adhoc().all.ping()
```

The following example demonstrates available keyword arguments when creating
a `HostManager` object.

```python
def test_uptime(ansible_adhoc):
    # take down the database
    ansible_adhoc(inventory='db1.example.com,', user='ec2-user',
        become=True, become_user='root').all.command('reboot')
```

The `HostManager` object returned by the `ansible_adhoc()` function provides
numerous ways of calling ansible modules against some, or all, of the
inventory. The following demonstates sample usage.

```python
def test_host_manager(ansible_adhoc):
    hosts = ansible_adhoc()

    # __getitem__
    hosts['all'].ping()
    hosts['localhost'].ping()

    # __getattr__
    hosts.all.ping()
    hosts.localhost.ping()

    # Supports [ansible host patterns](http://docs.ansible.com/ansible/latest/intro_patterns.html)
    hosts['webservers:!phoenix'].ping()  # all webservers that are not in phoenix
    hosts[0].ping()
    hosts[0:2].ping()

    assert 'one.example.com' in hosts

    assert hasattr(hosts, 'two.example.com')

    for a_host in hosts:
        a_host.ping()
```

#### Fixture `localhost`

The `localhost` fixture is a convenience fixture that surfaces
a `ModuleDispatcher` instance for ansible host running `pytest`. This is
convenient when using ansible modules that typically run on the local machine,
such as cloud modules (ec2, gce etc...).

```python
def test_do_something_cloudy(localhost, ansible_adhoc):
    """Deploy an ec2 instance using multiple fixtures."""
    params = dict(
        key_name='somekey',
        instance_type='t2.micro',
        image='ami-123456',
        wait=True,
        group='webserver',
        count=1,
        vpc_subnet_id='subnet-29e63245',
        assign_public_ip=True,
    )

    # Deploy an ec2 instance from localhost using the `ansible_adhoc` fixture
    ansible_adhoc(inventory='localhost,', connection='local').localhost.ec2(**params)

    # Deploy an ec2 instance from localhost using the `localhost` fixture
    localhost.ec2(**params)
```

#### Fixture `ansible_module`

The `ansible_module` fixture allows tests and fixtures to call [ansible
modules](http://docs.ansible.com/modules.html). Unlike the `ansible_adhoc`
fixture, this fixture only uses the options supplied to `pytest` at run time.

A very basic example demonstrating the ansible [`ping` module](http://docs.ansible.com/ping_module.html):

```python
def test_ping(ansible_module):
    ansible_module.ping()
```

A more involved example of updating the sshd configuration, and restarting the
service.

```python
def test_sshd_config(ansible_module):

    # update sshd MaxSessions
    contacted = ansible_module.lineinfile(
        dest="/etc/ssh/sshd_config",
        regexp="^#?MaxSessions .*",
        line="MaxSessions 150")
    )

    # assert desired outcome
    for (host, result) in contacted.items():
        assert 'failed' not in result, result['msg']
        assert 'changed' in result

    # restart sshd
    contacted = ansible_module.service(
        name="sshd",
        state="restarted"
    )

    # assert successful restart
    for (host, result) in contacted.items():
        assert 'changed' in result and result['changed']
        assert result['name'] == 'sshd'

    # do other stuff ...
```

#### Fixture `ansible_facts`

The `ansible_facts` fixture returns a JSON structure representing the system
facts for the associated inventory. Sample fact data is available in the
[ansible
documentation](http://docs.ansible.com/playbooks_variables.html#information-discovered-from-systems-facts).

Note, this fixture is provided for convenience and could easily be called using
`ansible_module.setup()`.

A systems facts can be useful when deciding whether to skip a test ...

```python
def test_something_with_amazon_ec2(ansible_facts):
    for facts in ansible_facts:
        if 'ec2.internal' != facts['ansible_domain']:
            pytest.skip("This test only applies to ec2 instances")

```

Additionally, since facts are just ansible modules, you could inspect the
contents of the `ec2_facts` module for greater granularity ...

```python

def test_terminate_us_east_1_instances(ansible_adhoc):

    for facts in ansible_adhoc().all.ec2_facts():
        if facts['ansible_ec2_placement_region'].startswith('us-east'):
            '''do some testing'''
```

#### Parameterizing with `pytest.mark.ansible`

Perhaps the `--ansible-inventory=<inventory>` includes many systems, but you
only wish to interact with a subset. The `pytest.mark.ansible` marker can be
used to modify the `pytest-ansible` command-line parameters for a single
test. Please note, the fixture `ansible_adhoc` is the prefer mechanism for
interacting with ansible inventory within tests.

For example, to interact with the local system, you would adjust the
`host_pattern` and `connection` parameters.

```python
@pytest.mark.ansible(host_pattern='local,', connection='local')
def test_copy_local(ansible_module):

    # create a file with random data
    contacted = ansible_module.copy(
        dest='/etc/motd',
        content='PyTest is amazing!',
        owner='root',
        group='root',
        mode='0644',
    )

    # assert only a single host was contacted
    assert len(contacted) == 1, \
        "Unexpected number of hosts contacted (%d != %d)" % \
        (1, len(contacted))

    assert 'local' in contacted

    # assert the copy module reported changes
    assert 'changed' in contacted['local']
    assert contacted['local']['changed']
```

Note, the parameters provided by `pytest.mark.ansible` will apply to all
class methods.

```python
@pytest.mark.ansible(host_pattern='local,', connection='local')
class Test_Local(object):
    def test_install(self, ansible_module):
        '''do some testing'''
    def test_template(self, ansible_module):
        '''do some testing'''
    def test_service(self, ansible_module):
        '''do some testing'''
```

#### Inspecting results

When using the `ansible_adhoc`, `localhost` or `ansible_module` fixtures, the
object returned will be an instance of class `AdHocResult`. The
`AdHocResult` class can be inspected as follows:

```python

def test_adhoc_result(ansible_adhoc):
    contacted = ansible_adhoc(inventory=my_inventory).command("date")

    # As a dictionary
    for (host, result) in contacted.items():
        assert result.is_successful, "Failed on host %s" % host
    for result in contacted.values():
        assert result.is_successful
    for host in contacted.keys():
        assert host in ['localhost', 'one.example.com']

    assert contacted.localhost.is_successful

    # As a list
    assert len(contacted) > 0
    assert 'localhost' in contacted

    # As an iterator
    for result in contacted:
        assert result.is_successful

    # With __getattr__
    assert contacted.localhost.is_successful

    # Or __gettem__
    assert contacted['localhost'].is_successful
```

Using the `AdHocResult` object provides ways to conveniently access results
for different hosts involved in the ansible adhoc command. Once the specific
host result is found, you may inspect the result of the ansible adhoc command
on that use by way of the `ModuleResult` interface. The `ModuleResult`
class represents the dictionary returned by the ansible module for a particular
host. The contents of the dictionary depend on the module called.

The `ModuleResult` interface provides some convenient proprerties to
determine the success of the module call. Examples are included below.

```python

def test_module_result(localhost):
    contacted = localhost.command("find /tmp")

    assert contacted.localhost.is_successful
    assert contacted.localhost.is_ok
    assert contacted.localhost.is_changed
    assert not contacted.localhost.is_failed

    contacted = localhost.shell("exit 1")
    assert contacted.localhost.is_failed
    assert not contacted.localhost.is_successful
```

The contents of the JSON returned by an ansible module differs from module to
module. For guidance, consult the documentation and examples for the specific
[ansible module](http://docs.ansible.com/modules_by_category.html).

#### Exception handling

If `ansible` is unable to connect to any inventory, an exception will be raised.

```python
@pytest.mark.ansible(inventory='unreachable.example.com,')
def test_shutdown(ansible_module):

    # attempt to ping a host that is down (or doesn't exist)
    pytest.raises(pytest_ansible.AnsibleHostUnreachable):
        ansible_module.ping()
```

Sometimes, only a single host is unreachable, and others will have properly
returned data. The following demonstrates how to catch the exception, and
inspect the results.

```python
@pytest.mark.ansible(inventory='good:bad')
def test_inventory_unreachable(ansible_module):
    exc_info = pytest.raises(pytest_ansible.AnsibleHostUnreachable, ansible_module.ping)
    (contacted, dark) = exc_info.value.results

    # inspect the JSON result...
    for (host, result) in contacted.items():
        assert result['ping'] == 'pong'

    for (host, result) in dark.items():
        assert result['failed'] == True
```
