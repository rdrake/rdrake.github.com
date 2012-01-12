---
layout: post
title: Adding Full-Text Search To Your Site With Xapian Omega
---
## Introduction ##

[Xapian](http://xapian.org/) is a full-text search library implemented in C++. One of its sub-projects is [Omega](http://xapian.org/docs/omega/). which provides a suite of scripts to index and search content with minimal effort.

This post assumes you have a static website up and running already. It details how to add full-text search capabilities to just about any static website.

## Installation ##

Xapian can be installed either system-wide or in a different location by configuring it with a different prefix. For a VPS system-wide will suffice. For a shared host, Xapian will need to be installed in the home directory.

Browse to the [Xapian Downloads page](http://xapian.org/download/) in order to fetch the latest version. You will need both the `xapian-core` and `xapian-omega` packages. The current version as of writing is 1.2.6. Alternatively you can use your distribution packages, however those tend to be out-of-date.

### Site-Wide Install ###

    wget http://oligarchy.co.uk/xapian/1.2.6/xapian-core-1.2.6.tar.gz
    tar -xzf xapian-core-1.2.6.tar.gz
    cd xapian-core-1.2.6
    ./configure
    make && sudo make install
    
    wget http://oligarchy.co.uk/xapian/1.2.6/xapian-omega-1.2.6.tar.gz
    tar -xzf xapian-omega-1.2.6.tar.gz
    cd xapian-omega-1.2.6
    ./configure
    make && sudo make install

The `xapian-bindings` and `Search::Xapian` packages are optional at this point. If you wish to make use of Xapian from languages other than C++, then you may wish to grab and build xapian-bindings in a similar fashion as above.

### Local Install ###

When installing locally, the only difference is the location where Xapian and Omega are installed. Change both `./configure` lines to read as follows.

    ./configure --prefix=/home/<username>

You must add $prefix/bin to your path before building Omega. You can do this by exporting on the command line, or by adding the following to your `~/.bashrc.`

    export PATH=$PATH:$prefix/bin

Where `$prefix` is the prefix specified to the configure script. If you receive an error about not being able to locate `xapian-config`, you did not modify your path correctly.

## Post-Install Configuration ##

By default the templates required by Omega are not installed. In order to install them, copy the templates from the source tarball to `$prefix/lib/xapian-omega/`.

    cp -R ~/Downloads/xapian-omega-1.2.6/templates $prefix/lib/xapian-omega/

Then Omega must be configured. In `$prefix/etc` you will find `omega.conf`. This file contains variables which tell Omega where to find its data, templates, cdb files, and where to write its logs to. This file must be edited to reflect our environment.

For system-wide installations, the defaults are fine. Be sure the directories exist.

    mkdir -p /var/lib/omega/data

And that the templates are correctly copied to `/var/lib/omega/data`.

    cp -R ~/Downloads/xapian-omega-1.2.6/templates /var/lib/omega

For local installations, change these values to reflect your environment. For instance,

    database_dir /home/<username>/omega/data
    template_dir /home/<username>/omega/templates
    log_dir      /home/<username>/omega/logs
    cdb_dir      /home/<username>/omega/cdb

Again be sure to create the specified directories.

    mkdir -p /home/<username>/omega/{data,templates,logs,cdb}

And copy over the templates as described above.

## Indexing Your Site ##

Now that Omega is properly configured, it is time to index your site. This is accomplished with the `omindex` script. `Omindex` recursively scans for filetypes it understands, applies filters to extract the text from these files, and indexes their contents automatically. It takes several arguments.

### Dependencies ###

Depending on which files you wish to index, various dependencies may be required. Some of the more important dependencies are xpdf for PDF files, antiword for Microsoft Word, and various unarchiving utilities for other formats. An up-to-date listing of existing filters and their dependencies is available on the [Omega Overview](http://xapian.org/docs/omega/overview.html) page.

### Using Omindex ###

The `omindex` command itself is straightforward. It requires at minimum the location to store the index, as well as the location to index.

    omindex --db $database_dir/mysite /path/to/my/site

You can also specify a URL. This URL is prepended to the path of file links in the search view. It defaults to the root of the site, or /. The following is the same as the previous command.

    omindex --db $database_dir/mysite --url / /path/to/my/site

The man page for omindex lists all arguments that it accepts.

    man omindex

## Making it Searchable ##

After the index is built, the search functionality can finally be added. Omega ships with a CGI application which lives at `$prefix/lib/xapian-omega/bin/omega`. This can be copied to your application’s `cgi-bin` and executed via web server.

If you wish to test out your application’s search now, you can execute the CGI application from the command line. Note, you may not receive any results yet.

    $prefix/lib/xapian-omega/bin/omega "P=<query>"

### Can’t Open Database “Default” ###

By default, Omega tries to open up an index in `$database_dir` called “default.” There are several ways to fix this.

1. Rename your index to “default.”
1. Specify the `DB` parameter when invoking the CGI application.
1. Create a file named “default” in `$database_dir` and point it to the location of your actual index.

The latter option is easily accomplished as follows.

    vim $database_dir/default

Add the following contents.

    auto $database_dir/mysite

Omega should now search your index without the `DB` parameter specified.

If all went well, you should now be able to invoke the omega application from inside of your browser.
