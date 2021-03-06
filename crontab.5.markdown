% CRONTAB(5)
%
% 20 Nov 2019

NAME
====
crontab - table of cron jobs

DESCRIPTION
===========

This page describes the crontab format used by **dcron**. Where
possible, the format follows the conventions of other cron
implementations, such as Vixie cron.

You may wish to skip to the comprehensive section of **EXAMPLES**.

A crontab consists of one row for each job. Blank lines in the crontab
are ignored, as are lines which begin with a hash (#) and are used for
comments.

A job consists of a time specification, and command to run:

	* * * * * date

Additional attributes may be specified:

	* * * * * ID=myjob date

Shorthand time specifications can be given, in which case cron will
decide when the job runs:

	@hourly ID=myjob date

"System" crontabs (typically in /etc/cron.d; see the section **System
crontabs**) have an extended format which includes the user to execute
a job as.

	* * * * * matthew date

Time specification
------------------

The time specification fields are, in order:

field           values
------          -------
minute          0-59
hour            0-23
day of month    1-31
month           1-12; jan-dec
day of week     0-7; sun-sun

For any field, ranges can be given either numerically (eg. "1-3") or
using the text equivalents (eg. "mon-wed"). Sub-ranges can be
separated with a comma:

	# run every two hours between 11pm and 7am, and again at 8am
	0 23-7/2,8 * * * date

If you specify both a day in the month and a day of week, it will be
interpreted as the Nth such day in the month.

The alternative time specifications of @hourly, @daily, @weekly,
@monthly, and @yearly may also be used. In this case cron will keep a
log of when the job was last run, and use this to decide when to next
run the job. This means it is necessary to set the **ID=** attribute
to uniquely identify the job:

	# at least one hour has elapsed since it last ran
	@hourly ID=job1 date

A special time specification @reboot will begin the job on startup,
and no idenfier is required:

	@reboot date

System crontabs
---------------

"System" crontabs are files located in /etc/cron.d, and it is normal
for these files to be placed by a package manager.

The jobs run as a user specified on a per-job basis, following the
convention of other cron implementations:

	# switch to user "matthew" and run the command
	* * * * * matthew date

System crontabs remain are "owned" by root, which will be the email
recipient of output or errors.

Fine-grained control
--------------------

There's also a format available for finer-grained control of frequencies:

	# run whenever it's between 2-4 am, and at least one day (1d)
	# has elapsed since this job ran
	* 2-4 * * * ID=job2 FREQ=1d date

	# as before, but re-try every 10 minutes (10m) if my_command
	# exits with code 11 (EAGAIN)
	* 2-4 * * * ID=job3 FREQ=1d/10m my_command

These formats also update timestamp files, and so also require their jobs to be assigned
IDs.

Notice the technique used in the second example: jobs can exit with code 11 to
indicate they lacked the resources to run (for example, no network was
available), and so should be tried again after a brief delay. This works for
jobs using either @freq or FREQ=... formats; but the FREQ=.../10m syntax is the
only way to customize the length of the delay before re-trying.

Dependencies
------------

Jobs can be made to "depend" on, or wait until AFTER other jobs have
successfully completed. Consider the following crontab:

	* * * * * ID=job4 FREQ=1d first_command
	* * * * * ID=job5 FREQ=1h AFTER=job4/30m second_command

Here, whenever job5 is up to be run, if job4 is scheduled to run within the
next 30 minutes (30m), job5 will first wait for it to successfully complete.

(What if job4 doesn't successfully complete? If job4 returns with exit code
EAGAIN, job5 will continue to wait until job4 is retried---even if that won't
be within the hour. If job4 returns with any other non-zero exit code, job5
will be removed from the queue without running.)

Jobs can be told to wait for multiple other jobs, as follows:

	10 * * * * ID=job6 AFTER=job4/1h,job7 third_command

The waiting job6 doesn't care what order job4 and job7 complete in. If job6 comes
up to be re-scheduled (an hour later) while an earlier instance is still waiting, only a
single instance of job6 will remain in the queue. It will have all of its
"waiting flags" reset: so each of job7 and job4 (supposing again that job4 would run within the
next 1h) will again have to complete before job6 will run.

If a job waits on a @reboot or @noauto job, the target job being waited on will
also be scheduled to run. This technique can be used to have a common job scheduled as @noauto
that several other jobs depend on (and so call as a subroutine).

	# don't ever schedule this job on its own; only run it when it's triggered
	# as a "dependency" of another job (see below), or when the user explicitly
	# requests it through the "cron.update" file (see crond(8))
	@noauto ID=namedjob date

Command execution
-----------------

The command portion of a cron job is run with `/bin/sh -c ...` and may
therefore contain any valid Bourne shell command. A common practice is to
prefix your command with **exec** to keep the process table uncluttered. It is
also common to redirect job output to a file or to /dev/null. If you do not,
and the command generates output on stdout or stderr, that output will be
mailed to the local user whose crontab the job comes from. If you have crontabs
for special users, such as uucp, who can't receive local mail, you may want to
create mail aliases for them or adjust this behavior. (See crond(8) for details
how to adjust it.)

Whenever jobs return an exit code that's neither 0 nor 11 (EAGAIN), that event
will be logged, regardless of whether any stdout or stderr is generated. The job's
timestamp will also be updated, and it won't be run again until it would next
be normally scheduled. Any jobs waiting on the failed job will be canceled; they
won't be run until they're next scheduled.

EXAMPLES
========

Examples of regular user's crontab entries:

	# MIN HOUR DAY MONTH DAYOFWEEK	COMMAND

	# run `date` at 6:10 am every day
	10 6 * * * date

	# run every two hours at the top of the hour
	0 */2 * * * date

	# run every two hours between 11 pm and 7 am, and again at 8 am
	0 23-7/2,8 * * * date

	# run at 4:00 am on January 1st
	0 4 1 jan * date

	# run every day at 11 am, appending all output to a file
	0 11 * * * date >> /var/log/date-output 2>&1

	# schedule this job only once, when crond starts up
	@reboot date

	# at least one hour has elapsed since it last ran
	@hourly ID=job1 date

To request the last Monday, etc. in a month, ask for the "5th"
one. This will always match the last Monday, etc., even if there are
only four Mondays in the month:

	# run at 11 am on the first and last Mon, Tue, Wed of each month
	0 11 1,5 * mon-wed date

When the fourth Monday in a month is the last, it will match against
both the "4th" and the "5th" (it will only run once if both are
specified).

NOTES
=====
Unlike other cron daemons, this crond/crontab package doesn't try to do
everything under the sun. It doesn't try to keep track of user's preferred
shells; that would require special-casing users with no login shell. Instead,
it just runs all commands using `/bin/sh`. (Commands can of course be script
files written in any shell you like.)

Nor does it do any special environment handling. A shell script is
better-suited to doing that than a cron daemon. This cron daemon sets up only
four environment variables: USER, LOGNAME, HOME, and SHELL.

SEE ALSO
========
**crontab**(1)
**crond**(8)

AUTHORS
=======
Matthew Dillon (dillon@apollo.backplane.com): original developer
James Pryor (dubiousjim@gmail.com): current developer
Mark Hills (mark@xwax.org): contributor
