# rbenv rehash

I recently had an issue when running the rbenv command `rehash`. This command is useful to, after installing a new version of ruby or a new gem that adds some commands, refresh the rbenv executable shims.

Sometimes, though, you'll come across an error where it cannot rehash, i.e., `rbenv: cannot rehash: /some/path/to/.rbenv/shims/.rbenv-shim exists`.

The solution is simple: Just remove the offending file.

Why this happens is, also, thankfully, simple: Sometimes things go wrong and the temporary files are just not cleaned up by the previous `rehash` command.
