# Project Structure
Bobtail has two main packages. The core reactive primitives and utility functions are defined in the 
[`bobtail-rx` package](github.com/bobtail-dev/bobtail-rx). This functionality is included
and re-exported as part of the [`bobtail` package](github.com/bobtail-dev/bobtail), which
extends it with the `rxt` templating DSL and utility functions.

# Issues

Any bugs or feature requests can always be reported on the 
[`bobtail` issues page](github.com/bobtail-dev/bobtail/issues). If you know the bug or FR
you're reporting is related to the core framework rather than the templating extension,
you can instead report the issue on the `bobtail-rx` project.

# Making changes

## Environment setup

After forking and cloning the project, run `npm install`. To test your changes,
run `npm test`. To build your changes, run `npm run build`.

### Windows users

If you are using Windows for your development environment, you'll need to use the
[Linux Subsystem for Windows](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide) 
or, alternatively, a Bash emulator like Cygwin.

## Committing changes
Before committing code, you should always run `npm run build` and add the `dist/`
folder as part of your commit. (Yes, this should be a pre-commit hook). Once you've
committed and pushed to your fork, you can open a pull request on the parent project.

## Unit tests and CI

Bobtail is an extensively tested framework. Unit tests are written in 
[Jasmine](https://jasmine.github.io/). The `bobtail` package's
unit tests (which cover the `rxt` templating code only) are executed using 
[Karma](https://karma-runner.github.io/1.0/index.html). `bobtail-rx`'s unit tests (covering
everything else) are also written in Jasmine, but currently only executed in NodeJS. Adding
Karma support to execute these tests in the browser is a priority. We use 
[Travis](https://travis-ci.org/) for continuous integration, which you should enable for
your forked copy. 

New code should always be unit tested, and bug fixes should ship with a unit test
that reproduces the bug being solved.

## Documentation

The documentation (which is to say, this website) is written in Markdown as part of 
the [`botail-dev.github.io`](github.com/bobtail-dev/botail-dev.github.io) project.
Documentation improvements and contributions are always welcome!

# Contributors
Regular contributors are encouraged to join the 
[bobtail-dev Github organization](github.com/bobtail-dev).

