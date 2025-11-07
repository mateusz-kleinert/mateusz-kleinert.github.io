# Managing Python environments with `pyenv`

Managing Python environments is crucial for maintaining clean and reproducible development setups. The `pyenv` tool streamlines installing multiple Python versions, switching between them, and managing virtual environments, helping developers avoid conflicts and dependency issues between projects.

## Why managing Python environments matters?

Different projects often depend on different Python versions and packages, which can quickly lead to conflicts if managed system-wide. Using `pyenv`, developers can:

- Install and use multiple Python versions side by side.
- Specify which Python version to use globally or per project to avoid version conflicts.
- Create isolated environments, ensuring reproducibility and consistency for all project dependencies.

## Installation

For installation instruction the best approach is to check the [official repository](https://github.com/pyenv/pyenv?tab=readme-ov-file#installation). E.g.: on MacOS it's as simple as running:

```
brew update
brew install pyenv
```

## Common use cases

### Installing a new Python version

To install a new version of Python with `pyenv`, use:

```
pyenv install <version>
```

Replace `<version>` with the desired version. This command downloads and compiles Python, installing it under `~/.pyenv/versions`.

### Switching between Python versions

`pyenv` can switch Python versions in three main ways:

- **Global version (default)**:

```
pyenv global <version>
```

Sets the default Python version for all shells.

- **Local version (per-directory/project)**:

```
pyenv local <version>
```

Creates or updates a `.python-version` file in the current directory, switching Python only for that directory tree.

- **Shell version (temporary, per session)**:

```
pyenv shell <version>
```

Switches the Python version only for the current shell session.


### Creating and using virtual environments

`pyenv` integrates with the `pyenv-virtualenv` plugin, allowing easy management of virtual environments based on any installed Python version:

- **Create a new virtual environment**:

```
pyenv virtualenv <version> <name>
```

This creates a new virtual environment called `<name>` using Python version `<version>`.

- **Activate the environment**:

```
pyenv activate <name>
```

This switches to the virtual environment, automatically adjusting `PATH` and Python interpreter.

- **Deactivate the environment**:

```
pyenv deactivate
```

Leaves the virtual environment.


## Minimalistic workflow

1. Install Python 3.13.0
2. Set it globally
3. Create a project folder called `project`, move into it, and set a local version of Python to 3.10.7
4. Create a virtual environment called `project-venv`
5. Activate the `project-venv` environment
6. Once the work is complete deactivate the virtual environment

```
pyenv install 3.13.0
pyenv global 3.13.0

mkdir project
cd project
pyenv local 3.10.7

pyenv virtualenv 3.10.7 project-venv
pyenv activate project-venv

// project's work

pyenv deactivate
```

This setup ensures your project uses the intended Python version and dependencies, independent of system or other project settings.

### Conclusion

`pyenv` is an essential tool for Python developers needing to maintain multiple Python versions and manage isolated environments. Its streamlined commands, flexibility, and integration with other tools like `pyenv-virtualenv` make it invaluable for both beginners and advanced users—and especially in complex development or AI workflows requiring strict dependency controls.
