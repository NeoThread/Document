pyenv
-----

#### Installation

* git clone https://github.com/yyuu/pyenv.git ~/.pyenv
* echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
* echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
* echo 'eval "$(pyenv init -)"' >> ~/.zshrc
* exec $SHELL
* pyenv install 2.7.8 # or env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install 3.4.3
* pyenv uninstall

#### Installation (+ Virtualenv)

* git clone https://github.com/yyuu/pyenv-virtualenv.git ~/.pyenv/plugins/pyenv-virtualenv
* echo 'eval "$(pyenv init -)"' >> ~/.zshrc
* echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.zshrc
* exec $SHELL

#### Commands

* List all commands
	- pyenv commands
* Install Python
	- pyenv install 2.7.8
*	Set global Python version
	- pyenv global 2.7.6
* Set a local application-specific Python version
	- pyenv local 2.7.4 3.3.3
	- pyenv local --unset #unset
* Get a local application-specific Python version
	- pyenv local
* Get versions
	- python versions
* Create Virtualenv
	- pyenv virtualenv 2.7.10 my-virtual-env-2.7.10
	- pyenv virtualenv my-virtual-env-2.7.10 # from current
	- pyenv uninstall my-virtual-env-2.7.10
* List virtualenvs
	- pyenv virtualenvs
* Export plugins
	- pip freeze > requirement_file
* Import plugins
	- pip install -r requirement_file
