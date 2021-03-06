=========
fh-fablib
=========

Usage
=====

1. Install `pipx <https://pipxproject.github.io/pipx/>`__
2. Install fh-fablib

   a. ``pipx install fh_fablib`` if you're happy with the packaged version
   b. ``pipx install --editable git+ssh://git@github.com/feinheit/fh-fablib.git@main#egg=fh_fablib`` otherwise

3. Add a ``fabfile.py`` to your project. A minimal example follows:

   .. code-block:: python

       from pathlib import Path

       import fh_fablib as fl

       fl.require("1.0.20200824")
       fl.config.update(base=Path(__file__).parent, host="www-data@feinheit06.nine.ch")
       fl.config.update(domain="example.com", branch="main", remote="production")

       ns = fl.Collection(*fl.GENERAL, *fl.NINE)

4. Run ``fab --list`` to get a list of commands.

Loading the ``fh_fablib`` module automatically creates
``.git/hooks/pre-commit`` which runs ``fab check`` before each commit.


Configuration values
====================

- ``app = "app"``: Name of primary Django app containing settings, assets etc.
- ``base``: ``pathlib.Path`` object pointing to the base dir of the project.
- ``branch``: Branch containing code to be deployed.
- ``domain``: Primary domain of website. The database name and cache key
  prefix are derived from this value.
- ``host``: SSH connection string (``username@server``)
- ``remote``: git remote name for the server. Only used for the
  ``fetch`` task.


Adding or overriding bundled tasks
==================================

For the sake of an example, suppose that additional processes should be
restarted after deployment. A custom ``deploy`` task follows:

.. code-block:: python

    # ... continuing the fabfile above

    @fl.task
    def deploy(ctx):
        """Deploy once 🔥"""
        fl.deploy(ctx)  # Reuse
        with fl.Connection(fl.config.host) as conn:
            fl.run(conn, "systemctl --user restart other.service")

    ns.add_task(deploy)

.. note::

   Instead of making existing tasks more flexible or configurable it's
   preferable to contribute better building blocks resp. to improve
   existing buildings blocks to make it easier to build customized tasks
   inside projects. E.g. if you want to ``fmt`` additional paths it's
   better to build your own ``fmt`` task and not add configuration
   variables to the ``config`` dictionary.


Multiple environments
=====================

If you need multiple environments, add tasks which only update
``fl.config`` as follows:

.. code-block:: python

    from pathlib import Path

    import fh_fablib as fl

    fl.require("1.0.20200824")
    fl.config.update(base=Path(__file__).parent, host="www-data@feinheit06.nine.ch")

    @fl.task(aliases=["p"])
    def production(ctx):
        fl.config.update(domain="example.com", branch="master", remote="production")


    @fl.task(aliases=["s"])
    def stage(ctx):
        fl.config.update(domain="stage.example.com", branch="develop", remote="stage")

    ns = fl.Collection(*fl.GENERAL, *fl.NINE, production, stage)

Now, ``fab production pull-db``, ``fab stage deploy`` and friends should
work as expected.


Available tasks
===============

``fh_fablib.GENERAL``
~~~~~~~~~~~~~~~~~~~~~

- ``bitbucket``: Create a repository on Bitbucket and push the code
- ``check``: Check the coding style
- ``cm``: Compile the translation catalogs
- ``deploy``: Deploy once 🔥
- ``dev``: Run the development server for the frontend and backend
- ``fetch``: Ensure a remote exists for the server and fetch
- ``fmt``: Format the code
- ``freeze``: Freeze the virtualenv's state
- ``github``: Create a repository on GitHub and push the code
- ``local``: Local environment setup
- ``mm``: Update the translation catalogs
- ``pull-db``: Pull a local copy of the remote DB and reset all passwords
- ``update``: Update virtualenv and node_modules to match the lockfiles
- ``upgrade``: Re-create the virtualenv with newest versions of all libraries


``fh_fablib.NINE``
~~~~~~~~~~~~~~~~~~

- ``nine``: Run all nine🌟 setup tasks in order
- ``nine-alias-add``: Add aliasses to a nine-manage-vhost virtual host
- ``nine-alias-remove``: Remove aliasses from a nine-manage-vhost virtual host
- ``nine-checkout``: Checkout the repository on the server
- ``nine-db-dotenv``: Create a database and initialize the .env.
  Currently assumes that the shell user has superuser rights (either
  through ``PGUSER`` and ``PGPASSWORD`` environment variables or through
  peer authentication)
- ``nine-disable``: Disable a virtual host, dump and remove the DB and
  stop the gunicorn@ unit
- ``nine-ssl``: Activate SSL
- ``nine-unit``: Start and enable a gunicorn@ unit
- ``nine-venv``: Create a venv and install packages from requirements.txt
- ``nine-vhost``: Create a virtual host using nine-manage-vhosts


Building blocks
===============

The following functions may be used to build your own tasks. They cannot
be executed directly from the command line.

Running commands
~~~~~~~~~~~~~~~~~

- ``run(c, ...)``: Wrapper around ``Context.run`` or ``Connection.run``
  which always sets a few useful arguments (``echo=True``, ``pty= True``
  and ``replace_env=False`` at the time of writing)


Checks
~~~~~~

- ``_check_flake8(ctx)``: Run ``venv/bin/flake8``
- ``_check_django(ctx)``: Run Django's checks
- ``_check_prettier(ctx)``: Check whether the frontend code conforms to
  prettier's formatting
- ``_check_eslint(ctx)``: Run ESLint
- ``_check_branch(ctx)``: Terminates if checked out branch does not
  match configuration.


Formatters
~~~~~~~~~~

- ``_fmt_isort(ctx)``: Run ``isort``
- ``_fmt_black(ctx)``: Run ``black``
- ``_fmt_prettier(ctx)``: Run ``prettier``
- ``_fmt_tox_style(ctx)``: Run ``tox -e style``


Helpers
~~~~~~~

- ``_local_env(path=".env")``: ``speckenv.env`` for a local env file
- ``_srv_env(conn, path)``: ``speckenv.env`` for a remote env file
- ``_python3()``: Return the path of a Python 3 executable. Prefers
  newer Python versions.
- ``_local_dotenv_if_not_exists()``: Ensure a local ``.env`` with a few
  default values exists. Does nothing if ``.env`` exists already.
- ``_local_dbname()``: Ensure a local ``.env`` exists and return the
  database name.
- ``_dbname_from_dsn(dsn)``: Extract the database name from a DSN.
- ``_dbname_from_domain(domain)``: Mangle the domain to produce a string
  suitable as a database name, database user and cache key prefix.
- ``_concurrently(ctx, jobs)``: Run a list of shell commands
  concurrently and wait for all of them to terminate (or Ctrl-C).
- ``_random_string(length, chars=None)``: Return a random string of
  length, suitable for generating secret keys etc.
- ``_reset_passwords(ctx)``: Set all user passwords to ``"password"``.
- ``require(version)``: Terminate if fh_fablib is older.
- ``terminate(msg)``: Terminate processing with an error message.


Recommended configuration files
===============================

``.editorconfig``
~~~~~~~~~~~~~~~~~

::

    # top-most EditorConfig file
    root = true

    [*]
    end_of_line = lf
    insert_final_newline = true
    charset = utf-8
    trim_trailing_whitespace = true
    indent_style = space
    indent_size = 4

    [*.{html,js,scss}]
    indent_size = 2


``.eslintrc.js``
~~~~~~~~~~~~~~~~

::

    module.exports = {
      env: {
        browser: true,
        es2020: true,
        node: true,
      },
      extends: [
        "eslint:recommended",
        "prettier",
        "preact",
        // "prettier/react",
        // "plugin:react/recommended",
      ],
      parser: "babel-eslint",
      parserOptions: {
        ecmaFeatures: {
          experimentalObjectRestSpread: true,
          jsx: true,
        },
        sourceType: "module",
      },
      plugins: [
        // "react",
        // "react-hooks",
      ],
      rules: {
        "no-unused-vars": [
          "error",
          {
            argsIgnorePattern: "^_",
            varsIgnorePattern: "React|Fragment|h|^_",
          },
        ],
        // "react/prop-types": "off",
        // "react/display-name": "off",
        // "react-hooks/rules-of-hooks": "warn", // Checks rules of Hooks
        // "react-hooks/exhaustive-deps": "warn", // Checks effect dependencies
      },
      settings: {
        react: {
          version: "detect",
        },
      },
    }


``setup.cfg``
~~~~~~~~~~~~~

::

    [flake8]
    exclude=venv,build,docs,.tox,migrate,migrations,node_modules
    ignore=E203,W503
    max-line-length=88
    max-complexity=10


``package.json``
~~~~~~~~~~~~~~~~

::

    {
      "name": "feinheit.ch",
      "description": "feinheit",
      "version": "0.0.1",
      "private": true,
      "dependencies": {
        "babel-eslint": "^10.0.3",
        "eslint": "^7.7.0",
        "eslint-config-prettier": "^6.11.0",
        "fh-webpack-config": "^1.0.7",
        "prettier": "^2.1.1"
      },
      "eslintIgnore": [
        "app/static/app/lib/",
        "app/static/app/plugin_buttons.js"
      ]
    }


``webpack.config.js``
~~~~~~~~~~~~~~~~~~~~~

::

    const merge = require("webpack-merge")
    const config = require("fh-webpack-config")

    module.exports = merge.smart(
      config.commonConfig,
      // config.preactConfig,
      // config.reactConfig,
      config.chunkSplittingConfig
    )
