---
layout: post
title:  "Jekyll 配置笔记"
date:   2019-10-22 23:39:36  +0800
categories: jekyll
---

    使用 Jekyll 部署一个 GitHub Pages。

## 本地环境

* 操作系统：Windows 10
* 开发工具：Visual Studio Code

## 过程

### 下载 RubyInstaller

https://rubyinstaller.org/downloads/

### 命令行操作

```bat
:: 修改镜像源，必须操作
PS C:\Users\feilong\repo> gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/

PS C:\Users\feilong\repo> gem sources -l
*** CURRENT SOURCES ***

http://gems.ruby-china.com/

:: 这样你不用改你的 Gemfile 的 source
PS C:\Users\feilong\repo> bundle config mirror.https://rubygems.org https://gems.ruby-china.com

PS C:\Users\feilong\repo> gem install jekyll bundler

PS C:\Users\feilong\repo> jekyll -v
jekyll 4.0.0

:: 创建博客文件模板
PS C:\Users\feilong\repo> jekyll new vflong.github.io
Running bundle install in C:/Users/feilong/repo/vflong.github.io... 
  Bundler: Fetching gem metadata from https://gems.ruby-china.com/...............
  Bundler: Fetching gem metadata from https://gems.ruby-china.com/..
  Bundler: Resolving dependencies...
  Bundler: Using public_suffix 4.0.1
  Bundler: Using addressable 2.7.0
  Bundler: Using bundler 2.0.2
  Bundler: Using colorator 1.1.0
  Bundler: Using concurrent-ruby 1.1.5
  Bundler: Using eventmachine 1.2.7 (x64-mingw32)
  Bundler: Using http_parser.rb 0.6.0
  Bundler: Using em-websocket 0.5.1
  Bundler: Using ffi 1.11.1 (x64-mingw32)
  Bundler: Using forwardable-extended 2.6.0
  Bundler: Using i18n 1.7.0
  Bundler: Using sassc 2.2.1 (x64-mingw32)
  Bundler: Using jekyll-sass-converter 2.0.1
  Bundler: Using rb-fsevent 0.10.3
  Bundler: Using rb-inotify 0.10.0
  Bundler: Using listen 3.2.0
  Bundler: Using jekyll-watch 2.2.1
  Bundler: Using kramdown 2.1.0
  Bundler: Using kramdown-parser-gfm 1.1.0
  Bundler: Using liquid 4.0.3
  Bundler: Using mercenary 0.3.6
  Bundler: Using pathutil 0.16.2
  Bundler: Using rouge 3.12.0
  Bundler: Using safe_yaml 1.0.5
  Bundler: Using unicode-display_width 1.6.0
  Bundler: Using terminal-table 1.8.0
  Bundler: Using jekyll 4.0.0
  Bundler: Using jekyll-feed 0.12.1
  Bundler: Using jekyll-seo-tag 2.6.1
  Bundler: Using minima 2.5.1
  Bundler: Using thread_safe 0.3.6
  Bundler: Using tzinfo 1.2.5
  Bundler: Using tzinfo-data 1.2019.3
  Bundler: Using wdm 0.1.1
  Bundler: Bundle complete! 6 Gemfile dependencies, 34 gems now installed.
  Bundler: Use `bundle info [gemname]` to see where a bundled gem is installed.
New jekyll site installed in C:/Users/feilong/repo/vflong.github.io.

:: 本地启动
PS C:\Users\feilong\repo\vflong.github.io> bundle exec jekyll serve --port 8080
Configuration file: C:/Users/feilong/repo/vflong.github.io/_config.yml
            Source: C:/Users/feilong/repo/vflong.github.io
       Destination: C:/Users/feilong/repo/vflong.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
       Jekyll Feed: Generating feed for posts
                    done in 6.124 seconds.
 Auto-regeneration: enabled for 'C:/Users/feilong/repo/vflong.github.io'
    Server address: http://127.0.0.1:8080/
  Server running... press ctrl-c to stop.

:: 访问 http://127.0.0.1:8080/
```

## 补充

### Ubuntu 中配置开发环境

```bash
$ sudo apt install ruby gem -y
$ gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/

https://gems.ruby-china.com/ added to sources
https://rubygems.org/ removed from sources

$ gem sources -l
*** CURRENT SOURCES ***

https://gems.ruby-china.com/

$ sudo apt install jekyll bundler -y
$ bundle config mirror.https://rubygems.org https://gems.ruby-china.com
$ bundle update
$ bundle exec jekyll -v               
jekyll 4.0.0

$ bundle exec jekyll serve --port 8080
Configuration file: /home/feilong/repo/vflong.github.io/_config.yml
            Source: /home/feilong/repo/vflong.github.io
       Destination: /home/feilong/repo/vflong.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
       Jekyll Feed: Generating feed for posts
                    done in 0.494 seconds.
 Auto-regeneration: enabled for '/home/feilong/repo/vflong.github.io'
    Server address: http://127.0.0.1:8080/
  Server running... press ctrl-c to stop.

# 忽略这个报错信息，对开发环境无影响
$ jekyll -v                                                            
Traceback (most recent call last):
	13: from /usr/local/bin/jekyll:23:in `<main>'
	12: from /usr/local/bin/jekyll:23:in `load'
	11: from /var/lib/gems/2.5.0/gems/jekyll-4.0.0/exe/jekyll:11:in `<top (required)>'
	10: from /var/lib/gems/2.5.0/gems/jekyll-4.0.0/lib/jekyll/plugin_manager.rb:52:in `require_from_bundler'
	 9: from /usr/lib/ruby/vendor_ruby/bundler.rb:101:in `setup'
	 8: from /usr/lib/ruby/vendor_ruby/bundler.rb:135:in `definition'
	 7: from /usr/lib/ruby/vendor_ruby/bundler/definition.rb:35:in `build'
	 6: from /usr/lib/ruby/vendor_ruby/bundler/dsl.rb:13:in `evaluate'
	 5: from /usr/lib/ruby/vendor_ruby/bundler/dsl.rb:218:in `to_definition'
	 4: from /usr/lib/ruby/vendor_ruby/bundler/dsl.rb:218:in `new'
	 3: from /usr/lib/ruby/vendor_ruby/bundler/definition.rb:83:in `initialize'
	 2: from /usr/lib/ruby/vendor_ruby/bundler/definition.rb:83:in `new'
	 1: from /usr/lib/ruby/vendor_ruby/bundler/lockfile_parser.rb:95:in `initialize'
/usr/lib/ruby/vendor_ruby/bundler/lockfile_parser.rb:108:in `warn_for_outdated_bundler_version': You must use Bundler 2 or greater with this lockfile. (Bundler::LockfileError)
```

### Mac 中配置开发环境

```bash
> brew install ruby
> gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
> bundle config mirror.https://rubygems.org https://gems.ruby-china.com
> sudo gem install jekyll bundler
> bundle update
> bundle exec jekyll -v               
> bundle exec jekyll serve --port 8080
```

## 参考
* [creating-a-github-pages-site-with-jekyll](https://help.github.com/en/github/working-with-github-pages/creating-a-github-pages-site-with-jekyll)
* [jekyllrb](https://jekyllrb.com/docs/installation/windows/)
* [ruby-china](https://gems.ruby-china.com/)
