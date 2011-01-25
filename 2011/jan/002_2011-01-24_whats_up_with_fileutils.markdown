---
type: regular
state: draft
post-id: 2921208153
format: markdown
date: 2011-01-24T22:26:23-06:00
tags: ruby
title: what's up with FileUtils rm
---

Today, I was working on a ruby/rake script to deploy an application we've been working on the past few weeks. Part of this script required that we delete files within our deployment directory. This is what we encountered while cleaning up the script; we changed the remove from:

		require 'fileutils'
		include FileUtils
		
		sh "rm -rf *.txt"
to:

		require 'fileutils'
		include FileUtils
		
		rm_rf "*.txt"

Seems the same right? WRONG! Turns out the is a slight difference in the two. Using `sh "rm -rf *.txt"` will do as expected and delete all files in your current directory that end in `.txt`, however the latter doesn't remove any files from the directory. Swtiching the `rm_rf` to `rm` will actually raise:

		Errno::ENOENT: No such file or directory - *.txt

### why? ###

After digging through the [ruby](https://github.com/ruby/ruby) source code I discovered a few things about `FileUtils`. First off `FileUtils` does not do anything special if there is an asterix in the `path` being passed to `remove_file` (which is ultimately called from `rm` of any form). If you follow the `FileUtils` source you'll wind at a system call `unlink`. So what's different about `unlink`? Look at the [manual page](http://linux.die.net/man/2/unlink) you'll notice that the main difference between `unlink` and `rm` is that `rm` parses its arguments using `getopt`. Voila! Problem solved; the parse of `'*.txt'` will return the list of files to be removed. Which, in ruby, would be equivalent to:

		require 'fileutils'
		include FileUtils
		
		rm Dir.glob('*.txt')

### moral ###

Damn moral's are lame... remind me not to do this again. But seriously, be careful when using `FileUtils` because the implementation might not be exactly what you expect.
