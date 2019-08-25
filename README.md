# deployphp

deployphp is a perl program do deploy PHP source files to a production
environment (irony). It updates or copies PHP project files to a production
server taking care of several routine tasks to ensure consistency and
automation. Files can be backed up in the production server before updating.
New files can be minified and compressed to improve performance and to avoid
"in production changes/tampering". To use deployphp, a SSH key must be
set-up. This program does not work with passwords, only SSH public keys.

By default, only newer files are updated to the production server.
deployphp will generate a list of files in both production and local paths
and compare them for modification dates and times. Then newer files will be
sent using SSH to a temporary directory and then moved to the final place by
a script remotelly. If neither --minify nor --compress are used, files will
be left as copies from the local path to the production.

The --all option can be used to force deployphp to copy all files without
checking dates and times. This will make deployphp not work incrementally.
A file list can also be supplied in the command line after the args. This
will make deployphp work only with the files supplied. But dates and times
will still be checked and files will be copied only if they are newer.

deployphp will first search and parse the configuration file
$HOME/.deployphp and then the one on the current directory. Parameters of
the second config file will superseed the ones in the first config file.
Config files are in JSON format. See section bellow for a description of the
syntax. Both config files use the same syntax.

The list of files to update is built based on the target parameter informed
at runtime. The target can be a project or a single path. Both have to be
pre-defined in a config file or deployphp will fail. A path is one local
directory of files (including it's subdirectories) and a remote host. The
same local path or the same remote host can be used in different paths in a
config file. A project is a list of paths and or other projects. This way,
by calling deployphp informing a project, several paths will be updated at
once.

The update process is sequenced across all files (even if files are from
different paths). First all dates and times of files will be compared to
their remote dates and times. If --all is not used only newer files will be
updated.  Second backup will be performed in the remote host if --backup is
used (or backup is 1 in the config). Third files will be copied to a
temporary directory in the remote host but not updated. Fourth files are
updated by calling the update script that moves files to the final
destination, overwriting old ones. Fifth temporary files are removed. In
case of errors, update is halted.

## Features

* Fully automated upload of PHP source files to a production server

* Backup previous versions to enable roll-back

* Incremental upload by transfering only newer files

* Minify PHP sources, HTML, Javascript and CSS files

* JSON config file manages different directories and projects

## Configuration

deployphp config files have to be valid JSON. The basic syntax is a JSON
hash containing one or more keys. Each key can be a project, a path or an
exclude list. The exclude list will be used to remove files from the update.
A exclude key in any config's file root will be global to all projects or
paths. Each element in the exclude list is a regexp exmpression that will
be tested against all files. Projects are a list of other projects or paths.
And paths are one local directory and one remote host.

This is an exmple of a deployphp configuration file:

```json
{
  "all":{
    "comment":"a project containing two paths",
    "list":[
      "server1",
      "server2"
    ],
    "backup":1
  },
  "server1":{
    "comment":"one path",
    "host":"server1.domain.com",
    "user":"root",
    "localdir":"/home/debian/php/project1",
    "targetdir":"/var/www/html",
    "bkpdir":"/var/www/bkp",
    "tmpdir":"/tmp",
    "ownergroup":"www-data:www-data",
    "minprecomment":"<?php /* DO NOT CHANGE FILES IN PRODUCTION! */ ?>",
    "minify": "1",
    "compress": "gzip",
    "backup": 0,
    "verbose": "1"
  },
  "server2":{
    "comment":"other path",
    ...
  },
  "exclude":[
    "deployphp", "^info.php$", "00about.txt", "Session.vim", "TODO",
    "\\.sw.$", "^.#", "~$", "\\.bz2$", "\\.gz$", "\\.pl$", "\\.sh$", "\\.htm$",
    "\\bCVS\\/", "\\.git\\/", "\\bdevel\\/"
  ]
}
```

The configuration file defines two paths (server1 and server2) and a project
(all). By invoking deployphp --target=all updates all files in server1 and
server2. Note that backup will be done even if the backup flag is 0 in
server1 because flags have higher precedence in the project "all" that
included the "server1" path.

The exclusion list is specially good for specifying a list of what not to
send to production, like backup files, swap files, development directories
and test files, etc.  

### Project Configuration Parameters

Configuration parameters for projects are:

#### comment (string)

Not parsed, used to insert a comment.

#### list (array)

Array of other projects or paths that will be included in the update.

#### exclude (array)

List of regular expressions to remove files from the update list.
Exclusions specified here will be used in included paths.
Default is "".

#### backup (bool)

Flag (0|1) to to backup or not. If 1, will force backup to included paths.

#### verbos (bool)

Flag (0|1) to show more messsages. If 1, will force verbose to included
paths.

### Path Configuration Parameters

Configuration parameters for paths are:

#### comment (string)

Not parsed, used to insert a comment.

#### host (string)

Fully qualified remote host.

#### user (string)

User name with SSH access without password to the production server and
the remote PATH to update/access all files. This user will be used in
the SSH commands issued to the production server. Default is "root".

#### targetdir (string)

Base directory on the production server where the PHP files will be
deployed. Files in subdirectories will be copied to the apropriate
path. Required argument. Default is "".

#### localdir (string)

Local base directory containing the PHP files. Subdirectories (unless
included in the exclusion list will be updated). Default is ".".

#### bkpdir (string)

Directory where the backup tar files will be written if --backup
is supplied. Default is "".

#### bkpmask (string)

Mask of the backup file created on the production server with the
backup of all files before updating. %T is substituted for the
current date and time and %U is substituted by the user.
Default is "backup_%T.tar.gz".

#### tmpdir (string)

Temporary directory where files will be uncompress on the production
server and removed after updating. The directory will be created if not
existent. Any pre-existing files in this directory can be overwritten.
Default is "/tmp/deployphp". 

#### ownergroup (string)

Owner and group of the project files on the production server.
Default is "www-data:www-data".

#### minify (bool)

If set PHP, HTML and CSS files will be minified removing comments,
whitespaces and indentation keeping internal strings intact (PHP
source code). Better results are achieved when all HTML inline code
is kept between ?> and <? tags. If files are minified, a comment
will be prepended to all PHP source code (minprecomment argument).
Default is off ("0"). [EXPERIMENTAL] Use with care and test first!

#### minprecomment (string)

If minify is set this comment will be prepended to all PHP source
code. The goal is the add a warning to the files in a for of
comment. The comment should be in the syntax <?php /* warning */ ?>
rather then //, or the file will be ignored by the interpreter.
[EXPERIMENTAL] Use with care and test first! Default is "".

#### compress (string)

Compress files using ALG algorythm on the production server. This is
usefull when the HTTP server can read and server the compressed files to
the client. This saves CPU and diskspace because all files will be pre
compressed. Compression can be gzip or bzip2.
[NOT IMPLEMENTED] [TODO] Default is off ("").

#### exclude (array)

List of regular expressions to remove files from the update list.
Default is "".

#### backup (bool)

Flag (0|1) to to backup or not.

#### verbos (bool)

Flag (0|1) to show more messsages.

## Examples

  deployphp --target=all -v

  deployphp --target=server1 --backup --all --minify --compress=gzip

## Prerequisites

deployphp requires:

* rsync (installed both locally and on the production server)

* SSH public keys for access to the production environments

* config files defining all projects and/or paths used

## TODO

* Implement file compression.

* Projects are not "stackable". Right now there is only one that instantiates all paths.

* Detect invalid comments (//) in the code and report it

## Bugs

* Compression is not working.

* Minification of PHP source sometimes render the result unusable

## Author

deployphp was developed by Rodrigo Antunes rorabr@github.com

