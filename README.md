# tv_grab_az_sdjson_sqlite / tv_meta_az_sd.py
XMLTV grabber for Schedules Direct JSON service, optimized for large numbers
of channels, with local fast caching of xmltv output, and IMDB support.
Also includes a Tvheadend metadata program for populating fanart for Kodi.

Place the tv_grab_az_sdjson_sqlite and tv_meta_az_sd.py in the same location
as existing grabber files (probably /usr/bin).

Overview
--------

The tv_grab_az_sdjson_sqlite program grabs schedule information from
SchedulesDirect and creates an xmltv file for use by PVRs (such as
Tvheadend and MythTV).

The tv_meta_az_sd.py integrates with "tvhmeta" in Tvheaded 4.3+ to
provide additional fanart for Kodi 18+. It needs to be put in the same
directory as Tvheadend's "tvhmeta" program. The "fanart" checkbox
in the Configuration->Recording->(expert) then needs to be ticked.
The Tvheadend server with then call tv_meta_az_sd.py at the approriate
time to populate fanart.

You need a SchedulesDirect account for these programs to work.
Information from SchedulesDirect is stored in a local SQLite database.
Generated information can also be cached locally to improve performance.

This program adds a number of extra features for updating the description with extra information.
This is useful because many GUIs do not readily display details about programmes such as season
and episode information or cast information.

Information about artwork is also downloaded and this can also be included in the description
as URL tags that are displayed by some clients.

See the options --update-description-with-all and --update-description-with-artwork for extra information.

Many of the options can be prefixed with "no-" in order to disable them, such as
"--update-all --no-update-description-with-premiere --no-update-description-with-live"

Abbreviated samples of descriptions are given below for a movie and a tv programme episode.

====

Rambo: First Blood (1982).

(1982). (R). ☆☆☆. Vietnam veteran...

Advisory: Adult Language, Adult Situations, Brief Nudity, Graphic Violence.

Cast: Sylvester Stallone, Richard Crenna...

Categories: Action, Feature Film, Movie.

====

How I Met Your Mother - S03E16 - Sandcastles in the Sand.

S3/9 E16/20. (2008-04-21). (TV-PG). Robin rekindles...

Cast: Josh Radnor, Jason Segel...

Categories: Episode, Series, Show, Sitcom.

====

Support for Multiple Lineups
----------------------------

Some countries require multiple SD channel lineups to populate all the
channels that can be received by one tuner.

To support this, the configuration file can be manually edited
to add multiple "lineup=" lines.

For example if your configuration file contained "lineup=XXX-1234-DEFAULT"
then you could alter it to be:
```conf
lineup=XXX-1234-DEFAULT
lineup=XXX-4567-DEFAULT
lineup=XXX-890A-DEFAULT
```
Then the three lineups are fetched and all the channels and programmes
from all three lineups are output in the same xmltv output file.

Alternatively the UI can be used with the --configure option to
add all lineups to the configuration file.

Local Caching of Formatted Data
-------------------------------

This is an experimental feature. That means the options may change or
be removed in future releases and the generated output has not been
rigorously tested.

Local caching of formatted programme data can be enabled.
This will cache xmltv output to a local server such as to Redis.
The sqlite database (containing details of downloaded data)
is separate from the xmltv-output formatted caching done
with redis.

This significantly reduces the time taken for subsequent runs when
fetching multiple days of listings.  To enable this you will need the
Perl module "CHI" and "CHI::Driver::Redis" and a working Redis server
running locally.

Caching can also allow the xmltv file to only contain changes,
rather than a complete listing via --cache-ignore-unchanged-programmes.
So, by default, SD can have several week's of data and an xmltv file
will say "news is on at 8pm on 1st April", and on the following day's
run it will say again "news is on at 8pm on 1st April", and so on for
several weeks. With --cache-ignore-unchanged-programmes, the xmltv file
will only contain the item once. This avoids the PVR needing to process
the same information repeatedly every day.

The cache can be used via the extra arguments:
"--cache-driver=Redis"

Other Perl CHI drivers can be used, but are not rigorously tested
such as:
"--cache-driver=File"

The first time the programme is run, the formatted output is cached,
so the run takes longer than normal as this information is saved.
On the second and subsequent runs, the cache is checked, and if the
formatted output is already cached, then it is retrieved.

With multiple days and hundreds of channels, this can significantly
reduce overhead.

To check that redis caching is occurring, you can use "redis-cli --stat"
to monitor keys/memory usage of the redis server. The server
must have enough space to hold the cached data (maxmemory setting
in the redis server's redis.conf). Since the server only uses as
much memory as it needs, it can be useful to set it to a high value
such as "maxmemory 1gb", and then monitor how much memory is actually
used over the course of a week.

Ensure that the redis.conf does _not_ have a "maxmemory-policy" of
"noeviction". For example, set it to "maxmemory-policy allkeys-lru".

If you need to erase all cached entries in Redis then you can use
"redis-cli flushdb".

File caching is an alternative to redis and can be used if you have a
fast filesystem.  Since file system does not natively support purging,
it is recommended to also add the purge option. This is selected via
"--cache-driver=File --cache-purge-expired".

* --cache-driver
Currently only "Redis" is tested and needs the extra Perl modules
installed. The "File" backend appears to work but requires a fast
drive/tmpfs to provide peak performance and File would need the
"--cache-purge-expired" option to ensure old entries are expired.
* --cache-namespace-extra=str
Extra text to use when generating a cache namespace.  This is used if
you run the programme multiple times and alter options that affect the
generated output. For example if you run it with
--update-description-with-icons on one run, and with
--no-update-description-with-icons on a second run then you should specify
a different caching namespace to avoid retrieving incorrect programme details.
* --cache-expiry=n
String specifying expiry to use. By default the cached programme data expires
after a few days to recover space and ensure listings are regenerated.
An example would be "--cache-expiry='10 days'"
* --cache-ignore-unchanged-programmes (experimental)
If the programme was already in the cache then do not include it in the
generated xmltv file. This avoids PVRs having to constantly reprocess
programmes that have not changed.
This option is experimental and may be removed/renamed in the future.
* --cache-purge-expired
Purge expired entries from the cache. This is _not_ needed for
Redis and is inefficient for many other cache backend drivers
(such as File) and not implemented at all in some other drivers.
* --cache-force-clear
Force the cache to be cleared of entries. Useful if there are
problems. See also --force-download.
* --cache-root-dir=str
For File backend, specify the root directory in to which cache
files are placed. Default is a per-user sub-directory under the
user's .xmltv directory.
* --cache-compression-threshold=n
Strings bigger than this threshold in bytes will be compressed
before storing in the cache. Default is 2048 bytes. Setting it
too low will waste CPU compressing/uncompressing, setting it
too high will mean nothing is compressed and the cache will
be larger.

IMDB Support
------------

Experimental IMDB support is included.

This feature adds details to the generated xmltv from IMDB such as
ratings for shows, URL information, etc. This is useful since
SchedulesDirect does not provide ratings for shows, only for movies,
and many of the URLs it provides for movies are outdated.

An example of the additional IMDB fields in the generated file:
```
<url >https://www.imdb.com/title/tt2350522</url>
<url >https://www.imdb.com/title/tt1219024</url>
<episode-num system = "imdb.com" >episode/tt2350522</episode-num>
<episode-num system = "imdb.com" >series/tt1219024</episode-num>
<star-rating system = "imdb" ><value >9/10</value></star-rating>
```
Note that many lookups will fail primarily because tv guides are full of
local shows, or have programmes that are old where there are discrepencies
on when the show was made or how it should be spelled (such as "Quincy",
"Quincy, M.E.", and "Quincy M.E."). Although we could handle more
special cases, that would unfortunately slow down the majority of lookups
that work.

It is recommended that the redis cache server is enabled to use IMDB
since it will allow caching of mapping information, see
"Local Caching" options such as "--cache-driver=Redis".

The IMDB support needs mysql server running since sqlite was too slow
for the large volumes of data that needs to be frequently imported.

As user root:
```
mysql
create database imdb character set utf8mb4;
create user 'imdb'@'%' identified by 'imdb';
grant all on imdb.* to 'imdb'@'%';
create user 'imdb'@'localhost' identified by 'imdb';
grant all on imdb.* to 'imdb'@'localhost';
grant file on *.* to 'imdb'@'localhost';
flush privileges;
```

The grabber needs to be run with the extra argument `--use-imdb`, with
additional configuration available if you choose to use a different
username, password, database, etc. (see below). A common additional
argument is "--prefer-imdb-rating" to prefer the IMDB ratings
over the SchedulesDirect ratings.

The IMDB datasets can be downloaded from https://www.imdb.com/interfaces/
You need the `title.basics.tsv.gz`, `title.episode.tsv.gz`, and
`title.ratings.tsv.gz` (see `--imdb-download` for option to automatically
download the files if we have write access to the mysql secure directory).

These files should be uncompressed and placed in to your mysql
"secure" directory. This is because modern mysql locks down
bulk loading of data.
`mysql -p -e "SHOW VARIABLES WHERE Variable_Name = 'secure_file_priv'"`

For me, this gives `/var/db/mysql_secure` or `/var/lib/mysql-files/` depending
on the server.

The files will be automatically loaded if the grabber has access to this directory.
If necessary then the directory can be specified with `--imdb-dir=xxx`, the default
is `/var/db/mysql_secure`.

The files will need to be periodically downloaded and they
will be automatically reloaded in to the database. See --imdb-download option.

* --use-imdb
Use IMDB tables to supplement information, such as ratings (see also --prefer-imdb-rating), URL information, etc.
* --imdb-db-type=s
Local db type for storing IMDB data, default mysql (sqlite was far too slow for the large volumes of data).
* --imdb-db=s
Local db connection information, default is imdb@localhost which connects to the database called imdb locally.
* --imdb-user=s
Local user to use, default imdb.
* --imdb-pass=s
Local password to use, default imdb
* --imdb-dir=s
MySQL secure directory containing the IMDB downloaded files. The grabber needs access to this directory.
Default is `/var/db/mysql_secure`.
* --imdb-download=s
Periodically download IMDB files. Options are "day", "week", "month", "quarter", "half", "year".
This will download in to the mysql secure directory (typically `/var/db/mysql_secure`) and
you need write access to this directory. This can be done via `setfacl`, or (for single user
systems), the tvheadend process could be given write access to the directory via
`chgrp tvheadend /var/db/mysql_secure && chmod g+rwx /var/db/mysql_secure`.
Since IMDB normally has details for future episodes/movies already, it's recommended
to not download more frequently than every month.

Extra Options
-------------

* --merge-split=n
Movies that are split in to two segments by news programmes will be automatically merged if the news is <=_n_ minutes long.
An example is "--merge-split=5" to remove programmes that are only five minutes long if they programmes either side have the same title.
* --artwork-max-width=n
Prefer artwork with a width no larger than _n_. If no artwork exists at that size then larger artwork may be used.
An example is "--artwork-max-width-720".
* --prefer-imdb-rating
If IMDB data is loaded and a rating exists for the programme then use it instead of the SchedulesDirect rating.
The IMDB ratings tend to have a larger range of values so more easily differentiate average and above-average movies.
* --channel-regex=regex and ---channel-exclude-regex=regex
When outputting xmltv file, only output details for channels matching/not matching the _regex_. This allows the user to run the grabber multiple times and output extra days for important channels. See also --channel-short-days-regex.
* --channel-short-days=n and --channel-short-days-exclude-regex=s
For channels matching the channel-short-days-exclude-regex, only the full programme guide, for
other channels, output only the "--channel-short-days" worth of days.
This allows you to use --days for "good" channels and "--channel-short-days" for channels that are not interesting,
thus saving memory on PVRs and clients.
Note that it is an "exclude" regex since it's likely you know the channels that are interesting
and all the other channels are then marked as boring.
For example: --channel-short-days=3 --channel-short-days-exclude-regex="BBC|Movie|PBS"
* --no-download
Do not download the schedule, use database cache only.
* --no-prune
Do not prune the database of old schedules. Useful if running multiple times.
* --no-channel-output
Do not output the channels information in the xmltv file. This can be useful if you have hundreds of
channels that never change since it avoids the PVR re-processing them on every run.
* --output-bad-channel-details=[never|always|week|month|quarter|half|year]
SchedulesDirect frequently has channels for which it no longer publishes guide data.
So we now output a list of these channels so end clients can be updated. Use this option
to avoid a detailed list of these bad channels being displayed.
The default is never.
* --force-download
Erase schedules from DB and force a complete download of data.
* --force-vacuum
The database is automatically vacuumed to reduce clutter and improve performance.
However, a vacuum can be forced with this option.
* --vacuum-frequency=n
Delta (in seconds) of how frequently a vacuum should occur.
Vacuuming can take a long time so is not done frequently.
Default is once every few months.
* --content-rating-order=s
Use a specific content rating order based on rating agency. For example "MPAA,VCHIP" will prefer those to all other rating systems.
This is useful since many movies have multiple ratings but many systems only take the first content rating.
The default is based on US ratings, with fallbacks to other countries.
* --use-category-for-keyword
Instead of outputting keyword tags, output category tags instead. Useful for programs that cannot parse keywords.
* --update-category-with-grabber
Add an xmltv tag to the programme's categories. This allows the user to schedule programmes that
have a category of 'xmltv' (and, say, a title of "Bob") and so avoid recording other OTA channels
which have provided EIT details that would otherwise match the recording rule.

Extra Options for Updating Description
--------------------------------------

These options alter the programme's description. This can be useful since
some frontends do not display programme year, age rating, season, episode,
star rating, or other information in some views.

* --update-description-with-all
Enable all the below "update-description-with-" options except for --update-description-with-title, --update-description-with-icons, --update-description-with-icons-basic, --update-description-with-icons-entity and --update-description-with-artwork.
* --update-description-with-credits
Add credits (actors) to description.
* --update-description-with-categories
Include categories in description.
* --update-description-with-year
Add year to description, for example year movie was made.
* --update-description-with-season-episode
Add season/episode information to description.
* --update-description-with-icons
Add icons to the description to indicate category.
Note that some databases (such as MySQL/MariaDB) have their own idea of utf-8 which defaults to only three-bytes not four-bytes (which they call utf8mb4), so they cannot store all icons correctly.
* --update-description-with-icons-basic
Similar to --update-description-with-icons but only include icons that fit in to basic character set so supported by older databases.
* --update-description-with-icons-entity
Similar to --update-description-with-icons but use html entities for encoding characters. This will only for some clients which cannot handle utf-8 in the server.
* --update-description-with-premiere
Add indicator that an episode is brand new to the description.
* --update-description-with-live
Add indicator that an episode is live to the description.
* --update-description-with-advisory
Add content advisories (such as adult language) to description.
* --update-description-with-artwork
Some clients allow artwork to be embedded in the description using html image URLs. This option enables this.
* --update-description-with-stars
Add star rating to description.
* --update-description-with-stars-color=s
Specify the color for the stars, either "white" or "black".
Default is "white" which uses an outline star (unicode U+2606). The alternative is "black" which uses a filled star (unicode U+2605).
* --update-description-with-rating
Add programme age rating in to description. For example "TV-PG".
* --update-description-with-title
Include title details inside the description. So the description will include "ProgTitle - S01E02 - Subtitle".
This is useful for easier recording regex in some programs.
This is not included in --update-description-with-all since many UIs do show season/episode information in their UI.
* --update-previously-shown-with-year
Movies often have details of the year they were made but some clients expect those details in the previously-shown field.
* --benchmark
Provide details of how long it took to generate files.

Extra Supported Tags and Features
---------------------------------

The program has a few features that are enabled without user configuration options:
* Prefer ttvdb episode numbers if they differ from Gracenote episode numbers. This is useful since most metadata lookups use ttvdb episode numbers which can often be different.
* Download artwork for programmes.
* Support multiple xmltv episode number formats include series/ and episode/ formats.
* Tag new showings as premiere so they can be recorded in tvheadend.

Extra Packages Required
-----------------------

A list of extra (potentially non-standard) packages you may need to install is below:

- DateTime::Duration::Parse (libtime-duration-parse-perl)
- DateTime::Format::SQLite (libdatetime-format-sqlite-perl)
- DBD::SQLite
- File::Homedir
- IPC::Run3 (libipc-run3-perl)
- JSON
- JSON::XS (libjson-xs-perl)
- LWP::Protocols::https (liblwp-protocol-https-perl)
- LWP::UserAgent::Determined
- URO::Escape::XS (liburi-escape-xs-perl)
- XMLTV

For local (advanced) caching you also need:
- CHI (libchi-perl)
- CHI::Driver::Redis (libchi-driver-redis-perl)
- A redis-server running on the local machine (redis-server)

The other modules used are typically already installed as part of xmltv.

Different distributions may have different dependencies but this will install dependencies
on some distributions:
```shell-script
sudo apt install xmltv libdatetime-format-sqlite-perl libipc-run3-perl libjson-xs-perl liblwp-protocol-https-perl liburi-escape-xs-perl libchi-perl libchi-driver-redis-perl
```

Some distributions might ship with fewer dependencies pre-installed as
part of the base system.  If you have an error stating "you may need
to install the ABC::DEF module" then you can normally install it using
`sudo apt install libabc-def-perl` (note the installed package name is
all lowercase and has `lib` prefix and `-perl` suffix). If this does
not work, you can normally find the package name using
`apt search libabc-def-perl` or `apt search ABC::DEF`.

Miscellaneous Information
-------------------------

In Kodi, to view the star rating you may have to change the font to "Arial Based"
in Settings/Interface Settings/Skin/Fonts, otherwise it may appear as square
blocks instead of stars.

The airdate may be incorrect for your country since it tends to be the
first worldwide airdate.

Total number of seasons is known to be sometimes incorrect when a new
series is released.

Programmes marked as "new/premiere" are based on information provided
to Schedules Direct by the networks.  So, a movie may be "new" because
it's the first time a network has purchased the rights to show the
movie.  This means a movie from 1970 may be marked as "new" on one
network but could be a "repeat" on a different network. Similarly,
some networks mark programmes as "new" based on the timeslot.  So a
programme could be "new" at 9pm and the same episode could also be
"new" the following day at 2am.


How Do You (Personally) Run This Program?
-----------------------------------------

I like to have long listings on channels I watch, and shorter listings
on all the other channels such as shopping.

For mythtv/mythweb, I like to have artwork injected in the description.

I used to use File caching backend since I have a reasonably fast filesystem
backed by SSD and it seems a bit faster than Redis on my system. However,
I now use Redis since it performs purging faster than File.

So I have a script that is run via crontab. It is similar to:

```shell-script
#! /bin/sh

# Interesting channels have this text in their name (separated by pipe symbol)
CHREGEX="BBC|Movie|Film|Sony"

COMMONARGS="--config-file sd.conf --merge-split=5 --artwork-max-width=720 --update-description-with-all --update-description-with-artwork --update-previously-shown-with-year --cache-driver=Redis --cache-ignore-unchanged-programmes --benchmark --cache-ignore-unchanged-programmes --no-channel-output --use-imdb --prefer-imdb-rating --imdb-download=month"
tv_grab_az_sdjson_sqlite --output out.xml --days 10 --channel-short-days=3 --channel-short-days-exclude-regex="$CHREGEX" $COMMONARGS
mythfilldatabase --file --xmlfile out.xml --sourceid 1
```

For tvheadend, I don't use artwork in the descriptions since its grid doesn't handle them very well:
```
tv_grab_az_sdjson_sqlite --output tvh.xml $COMMON_ARGS --no-update-description-with-artwork --cache-namespace-extra=tvh
nc -w 5 -U /home/hts/.hts/tvheadend/epggrab/xmltv.sock < tvh.xml
```

Most people don't need this level of complexity and can use the script
from within the PVR directly, perhaps adding appropriate "extra arguments"
such as "--artwork-max-width=720 --cache-driver=Redis".
