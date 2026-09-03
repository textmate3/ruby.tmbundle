# Notes from test suite modernization (2026-08-20)

Findings from modernizing this bundle's test suite

## Result

`Support/test/test_executable.rb` 16 tests, 26 assertions, 100% passing under Ruby 4.0.6, zero warnings, runnable from any working directory:

```sh
ruby Support/test/test_executable.rb
```

## What was broken and since when

One thing only: a `def __dir__` compatibility shim from the Ruby 1.8 era, written before Ruby 2.0 made `__dir__` a builtin. Two problems with it:

1. The builtin returns an absolute path (`File.dirname(File.realpath(__FILE__))`) and the shim returned `File.dirname(__FILE__)`, which is only as absolute as the invocation. So the suite only loaded when launched from `Support/test/`. In there the shim happened to produce a `./`-prefixed path, the one relative form `require` accepts since 1.9.2 removed `.` from the load path.
2. A top-level method definition lands on `Object`, so the shim shadowed the `Kernel#__dir__` builtin for every file in the process. Any _other_ code calling `__dir__` would have received the test file's directory.

Deleting the shim fixed both. Everything else passed unchanged.

## Why this suite matters

It covers `Support/lib/executable.rb`, the engine deciding how bundle commands invoke tools (RSpec runner, ⌘R, RuboCop and friends), with this precedence: `TM_*` environment variable override, then project binstub (`bin/rspec`), then `bundle exec` when a Gemfile names the tool, then plain `$PATH`. Plus `rvm` wrapper prefixing and an `rbenv-shim-without-gem` trap. A green run here is automated smoke coverage for the Ruby bundle's command invocation layer, overlapping the tier-1/tier-2 planned manual smoke tests.

## Observations

- This is 2013-era TextMate 2 code and it shows. Theer's environment isolation in `setup`/`teardown`, fake `$HOME` directories, fake `rvm` and `rbenv` installs as real fixtures under `Support/test/fixtures/`, and a precedence test built on `Dir.mktmpdir`. Nothing else needed modernizing.
- Test runs leave the working tree clean.
- The known `ruby.tmbundle` fossils are elsewhere and remain untouched: `vendor/rcodetools`' `xmpfilter.rb` calls `String#map`/`grep`/`inject`. These are Ruby 1.8 methods. See the downstream note about `ruby_modernization.md`. See also, the `Tests/` directory at the bundle root is deliberately-unparseable syntax-highlighting fixtures, not a runnable suite. This is excluded by name in `script/check_ruby_compatibility.rb`.
