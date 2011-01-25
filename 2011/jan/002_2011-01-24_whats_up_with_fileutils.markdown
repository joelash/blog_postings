---
type: regular
state: draft
post-id: 2921208153
format: markdown
date: 2011-01-24T22:26:23-06:00
tags: ruby
title: what's up with FileUtils rm_r
---

Today, while at work, I was working on a ruby/rake script to deploy an application. Part of the deployment required us to delete some files within a directory. In cleaning up some of the code we changed our code from:

		require 'fileutils'
		include FileUtils
		
		sh "rm -rf *.txt"
to:

		require 'fileutils'
		include FileUtils
		
		rm_rf "*.txt"

Seem the same right? WRONG! Turns our they aren't at all. Using `sh "rm -rf *.txt"` will actually delete all files in your current directory, however the latter doesn't actually delete anything; swtich the `rm_rf` to a simple `rm_r` and ruby will actually raise

		Errno::ENOENT: No such file or directory - *.txt

### why? ###

After digging through the [ruby](https://github.com/ruby/ruby) source code and discovered a few things about `FileUtils`. First off `FileUtils` does not do anything special if there is an asterix in the `path` being passed to `remove_file` (which is ultimately called from `rm` of any form). If you follow the `FileUtils` code you wind at a system call `unlink`. So what's different about `unlink`? Look at the [manual page](http://linux.die.net/man/2/unlink) you'll notice that the main difference between `unlink` and `rm` is that `rm` parses it's arguments using `getopt`. Voila! Problem solved; the special `getopt` is important for this.

### moral ###

Damn moral's are lame... remind me not to do this again. But seriously, be careful when using `FileUtils` because the implementation might not be exactly what you expect.
