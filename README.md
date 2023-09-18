## pyutil

This directory houses shared Python utilities.  All Python libraries live in the `everlaw/` directory and in the `everlaw.*` namespace.

Check out the Confluence documentation for more details: https://everlaw.atlassian.net/wiki/spaces/ENG/pages/635437823/pyutil+everlaw.+Python+packages.

### What's included?

Some of the libraries we currently have:

* `everlaw.commons`: Basic functionality used by other packages.  This should have minimal thirdparty dependencies and *no* internal dependencies.

* `everlaw.pypkg`: Package for managing our packages.

* `everlaw.wrappers.git`: Wrappers around `git` commands that manipulate structured data.  Meant to give an object-oriented interface to version control.

* `everlaw.versioning`: Tools for managing our software versions.

* `everlaw.infra`: Tools and scripts for managing our infrastructure (currently just a stub).

* `everlaw.testutil`: Helpers for unit and intergration testing (currently just a stub).

* `everlaw.thirdparty.*`: Helpers for interacting with thirdparty APIs.

NOTE: It's possible that this list is out of date.  For the most up-to-date list of packages, try this command:

    $ find pyutil/ -name pkg.yml

We also have some miscellaneous stuff:

* `setup-env`: A `source`able script that takes care of setting up a virtual environment and installing some of our basic packages (e.g., `everlaw.commons`).  This script is `source`d by `dev/env.sh`, which you should have already `source`d :^)

* `build-support/`: Tools and data for managing the development environment.

* `pom.xml`: A `maven`-compatible build target so that the rest of the Java codebase can declare dependencies on us.

### Dependencies

To install all necessary dependencies on your system, you can just apply
the 'python3.8' role locally.  That looks something like

    $ cd ansible/
    $ ansible-playbook \
        -i localhost \
        single.yml \
        -e role=python3.8 \
        -e current_stage=ami \
        --diff \
        -vv

For an up-to-date list of the current versions of Python we support, check
out `build-support/prelude`.

### Using this library

Quick start:

    $ source dev/env.sh
    $ pypkg --help

To import an object defined in this package from another Python script or interpreter, you'll need to make sure that script is operating inside our ["virtual environment"](https://docs.python.org/3/library/venv.html).  The easiest way to do that is to run

    $ source dev/env.sh
    $ pypkg install pyutil/everlaw/path/to/my/package -v
    $ # OR
    $ pypkg install everlaw.name.of.my.package -v

Then, you'll be able to do stuff like

```python
>>> import everlaw.wrappers.git as git
>>> git.current_ref()
develop
```

It's possible to make this process even easier in the future: see [this Trello card](https://trello.com/c/zKjxO2wL/26-add-an-easy-way-to-run-a-single-entrypoint-without-needing-a-wrapper).

#### Something's wrong...

Sometimes, the environment breaks such that even `pypkg --help` or other simple commands fail unexpectedly.  If 
that's the case, you can try to repair your environment with:

    $ pyutil/build-support/repair-venv
    $ source dev/env.sh

If that fails, here's how you rebuild your environment:

    $ pyutil/build-support/purge-venv
    $ source dev/env.sh

### Structure

Our packages are organized as ["namespace packages"](https://packaging.python.org/guides/packaging-namespace-packages/) (see [PEP 420](https://www.python.org/dev/peps/pep-0420/)).  In effect, this means that all of our packages can be imported as `everlaw.$something`.

Each package should live in a corresponding subdirectory of `pyutil/everlaw/`.  See [`everlaw.pypkg`](https://github.com/Everlaw/servers/tree/develop/pyutil/everlaw/pypkg) for more details on our internal structure.

### Tools

This package uses a few static analysis tools (e.g. testing, linting, type checking), all managed by `everlaw.pypkg`.  Check out [the documentation for that package](https://github.com/everlaw/servers/tree/develop/pyutil/everlaw/pypkg/) for details.

### Deploying Pyutil Code on Remote Instances

There are two parts to deploying pyutil code to remote instances: building the code / storing the artifacts in S3 and "installing" the code on the instance.

#### Building code and storing the artifacts in S3

There is a `StandardService` enum in the `everlaw.versioning_impls` package that defines which services can be managed using pyutil. Each member of the enum is a subclass of `VersionedService` and knows how to build and package itself into an artifact suitable for deployment. CircleCI builds and pushes new versions of all of the services defined in the `StandardService` enum whenever there are changes that would affect the artifacts. This process can also be done manually using the `build-standard-service` and `push-new-version` pyutil scripts. There are only two steps that need to be done to support building and packaging a pyutil package into a deployment artifact:
* Use the `make_pypkg_service` function to create a subclass of `VersionedService` for the pyutil package
* Add the new class to the `StandardService` enum
Once these steps are complete the `build-standard-service` and `push-new-version` scripts can be used to package the pyutil package into an artifact and store the artifact in S3.

#### Installing pyutil artifacts

The `pypkg-client` ansible role provides the `install-pypkg-from-s3` script, which can be used to install a pyutil package stored in S3. The role also provides several wrapper scripts (`with-venv` and `with-conda-venv`) that can be used to install the package in a virtual environment. Unless you specifically need to install the package globally you should always install the package in a virtual environment! The wrappers expect the virtual environment to have already been created, so make sure the environment is created before installing the package. The script needs to know the name, key, and version of the artifact that is going to be installed. The name is simply the name of the pyutil package (e.g. `everlaw.workers.clustering.cpu`). The key and version can be determined using the `query-version` script. 

Note that this script works by querying information about the servers repository from git, which means you need a way to compute this value with the source code checked out. For services tied to the web application, this is done by writing the output to a properties file at build-time (see https://github.com/Everlaw/servers/blob/develop/versioning/pom.xml). For services not tied to the web application, this can be done by creating an Ansible playbook that computes the version and key on the controller before running `install-pypkg-from-s3` on the remote host.

### Conventions

Let's codify things here as we agree on them :^)

##### naming conventions

In general, this project follows [PEP 8 naming conventions](https://www.python.org/dev/peps/pep-0008/#naming-conventions).  On top of that, here are some more guidelines:

* any name that conflicts with a builtin symbol should have a `_` appended, e.g.,

    ```python
    hash = compute_hash(...)   # bad, conflicts with builtins.hash
    hash_ = compute_hash(...)  # good!
    ```

##### default parameters

* make sure not to mutate default "reference" parameters, e.g.,

    ```python
    >>> def foo(bar: Dict[Any, Any] = {}) -> Any:
    ...     print(bar.get("baz", None))
    ...     bar["baz"] = "qux"
    ...
    >>> foo()
    None
    >>> foo()
    qux
    ```

    This can lead to weird, hard-to-track-down bugs.  In general, we should try to avoid mutating parameters.  If you really need to, these patterns should avoid the problem above:

    ```python
    >>> # Use Optional[T] types
    >>> def foo(bar: Optional[Dict[Any, Any]] = None) -> Any:
    ...     bar = bar or {}
    ...     bar["baz"] = "qux"
    ...
    >>> # Use *spread operator
    >>> def foo(bar: Dict[Any, Any] = {}) -> Any:
    ...     bar = {**bar, "baz": "qux"}  # our entries come _after_
    ...
    >>> # Use dict::copy()
    >>> def foo(bar: Dict[Any, Any] = {}) -> Any:
    ...     bar = bar.copy()  # it's a new reference here
    ...     bar["baz"] = "qux"
    ...
    ```

    Note that these fixes are only sufficient for avoiding mutation of **non-nested** dictionaries.  That is, reference-type keys and values will **not** be copied-by-value using either the "spread" or "::copy()" methods.

### Appendix: Tips and Tricks

#### Some helpful Bash aliases

Here are some helpful Bash aliases to add to your `~/.bashrc`:
```bash
alias pyutil="source \$(git rev-parse --show-toplevel)/dev/env.sh"
alias pyrepair="\$(git rev-parse --show-toplevel)/pyutil/build-support/repair-venv"
alias pypurge="\$(git rev-parse --show-toplevel)/pyutil/build-support/purge-venv"
alias pyrun="\$(git rev-parse --show-toplevel)/pyutil/bin/install-pypkg-deps-and-run"
```
# github-course
