# About VObject

VObject is intended to be a full-featured Python package for parsing and generating vCard and vCalendar files. It was originally developed in concert with the Open Source Application Foundation's Chandler project by Jeffrey Harris. Many thanks to [all the contributors](https://github.com/eventable/vobject/blob/master/ACKNOWLEDGEMENTS.txt) for their dedication and support. The project is currently being maintained by [Eventable](https://github.com/eventable) and [Sameen Karim](https://github.com/skarim).

Currently, iCalendar files are supported and well tested. vCard 3.0 files are supported, and all data should be imported, but only a few components are understood in a sophisticated way. The [Calendar Server](http://calendarserver.org/) team has added VAVAILABILITY support to VObject's iCalendar parsing. Please report bugs and issues directly on [GitHub](https://github.com/eventable/vobject/issues).

VObject is licensed under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0). [![License](https://img.shields.io/pypi/l/vobject.svg)](http://www.apache.org/licenses/LICENSE-2.0.html)

Useful scripts included with VObject:

* [ics_diff](https://github.com/eventable/vobject/blob/master/vobject/ics_diff.py): order is irrelevant in iCalendar files, return a diff of meaningful changes between icalendar files
* [change_tz](https://github.com/eventable/vobject/blob/master/vobject/change_tz.py): Take an iCalendar file with events in the wrong timezone, change all events or just UTC events into one of the timezones PyICU supports. Requires [PyICU](https://pypi.python.org/pypi/PyICU/).


# Installation [![PyPI version](https://badge.fury.io/py/vobject.svg)](https://pypi.python.org/pypi/vobject)

To install with [pip](https://pypi.python.org/pypi/pip), run:

```
pip install vobject
```


Or download the package and run:

```
python setup.py install
```

VObject requires Python 2.7 or higher, along with the [dateutil](https://pypi.python.org/pypi/python-dateutil/) and [six](https://pypi.python.org/pypi/six) packages.


# Running tests [![Build Status](https://travis-ci.org/eventable/vobject.svg?branch=master)](https://travis-ci.org/eventable/vobject)

To run all tests, use:

```
python tests/tests.py
```


# Usage

## iCalendar

#### Creating iCalendar objects

VObject has a basic datastructure for working with iCalendar-like
syntaxes.  Additionally, it defines specialized behaviors for many of
the commonly used iCalendar objects.

To create an object that already has a behavior defined, run:

```
>>> import vobject
>>> cal = vobject.newFromBehavior('vcalendar')
>>> cal.behavior
<class 'vobject.icalendar.VCalendar2_0'>
```

Convenience functions exist to create iCalendar and vCard objects:

```
>>> cal = vobject.iCalendar()
>>> cal.behavior
<class 'vobject.icalendar.VCalendar2_0'>
>>> card = vobject.vCard()
>>> card.behavior
<class 'vobject.vcard.VCard3_0'>
```

Once you have an object, you can use the add method to create
children:

```
>>> cal.add('vevent')
<VEVENT| []>
>>> cal.vevent.add('summary').value = "This is a note"
>>> cal.prettyPrint()
 VCALENDAR
    VEVENT
       SUMMARY: This is a note
```

Note that summary is a little different from vevent, it's a
ContentLine, not a Component.  It can't have children, and it has a
special value attribute.

ContentLines can also have parameters.  They can be accessed with
regular attribute names with _param appended:

```
>>> cal.vevent.summary.x_random_param = 'Random parameter'
>>> cal.prettyPrint()
 VCALENDAR
    VEVENT
       SUMMARY: This is a note
       params for  SUMMARY:
          X-RANDOM ['Random parameter']
```

There are a few things to note about this example

  * The underscore in x_random is converted to a dash (dashes are
    legal in iCalendar, underscores legal in Python)
  * X-RANDOM's value is a list.

If you want to access the full list of parameters, not just the first,
use &lt;paramname&gt;_paramlist:

```
>>> cal.vevent.summary.x_random_paramlist
['Random parameter']
>>> cal.vevent.summary.x_random_paramlist.append('Other param')
>>> cal.vevent.summary
<SUMMARY{'X-RANDOM': ['Random parameter', 'Other param']}This is a note>
```

Similar to parameters, If you want to access more than just the first child of a Component, you can access the full list of children of a given name by appending _list to the attribute name:

```
>>> cal.add('vevent').add('summary').value = "Second VEVENT"
>>> for ev in cal.vevent_list:
...     print ev.summary.value
This is a note
Second VEVENT
```

The interaction between the del operator and the hiding of the
underlying list is a little tricky, del cal.vevent and del
cal.vevent_list both delete all vevent children:

```
>>> first_ev = cal.vevent
>>> del cal.vevent
>>> cal
<VCALENDAR| []>
>>> cal.vevent = first_ev
```

VObject understands Python's datetime module and tzinfo classes.

```
>>> import datetime
>>> utc = vobject.icalendar.utc
>>> start = cal.vevent.add('dtstart')
>>> start.value = datetime.datetime(2006, 2, 16, tzinfo = utc)
>>> first_ev.prettyPrint()
     VEVENT
        DTSTART: 2006-02-16 00:00:00+00:00
        SUMMARY: This is a note
        params for  SUMMARY:
           X-RANDOM ['Random parameter', 'Other param']
```

Components and ContentLines have serialize methods:

```
>>> cal.vevent.add('uid').value = 'Sample UID'
>>> icalstream = cal.serialize()
>>> print icalstream
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//PYVOBJECT//NONSGML Version 1//EN
BEGIN:VEVENT
UID:Sample UID
DTSTART:20060216T000000Z
SUMMARY;X-RANDOM=Random parameter,Other param:This is a note
END:VEVENT
END:VCALENDAR
```

Observe that serializing adds missing required lines like version and
prodid.  A random UID would be generated, too, if one didn't exist.

If dtstart's tzinfo had been something other than UTC, an appropriate
vtimezone would be created for it.


#### Parsing iCalendar objects

To parse one top level component from an existing iCalendar stream or
string, use the readOne function:

```
>>> parsedCal = vobject.readOne(icalstream)
>>> parsedCal.vevent.dtstart.value
datetime.datetime(2006, 2, 16, 0, 0, tzinfo=tzutc())
```

Similarly, readComponents is a generator yielding one top level component at a time from a stream or string.

```
>>> vobject.readComponents(icalstream).next().vevent.dtstart.value
datetime.datetime(2006, 2, 16, 0, 0, tzinfo=tzutc())
```

More examples can be found in source code doctests.


## vCards

#### Creating vCard objects

Making vCards proceeds in much the same way. Note that the 'N' and 'FN'
attributes are required.

```
>>> j = vobject.vCard()
>>> j.add('n')
 <N{}    >
>>> j.n.value = vobject.vcard.Name( family='Harris', given='Jeffrey' )
>>> j.add('fn')
 <FN{}>
>>> j.fn.value ='Jeffrey Harris'
>>> j.add('email')
 <EMAIL{}>
>>> j.email.value = 'jeffrey@osafoundation.org'
>>> j.email.type_param = 'INTERNET'
>>> j.prettyPrint()
 VCARD
    EMAIL: jeffrey@osafoundation.org
    params for  EMAIL:
       TYPE ['INTERNET']
    FN: Jeffrey Harris
    N:  Jeffrey  Harris
```

serializing will add any required computable attributes (like 'VERSION')

```
>>> j.serialize()
'BEGIN:VCARD\r\nVERSION:3.0\r\nEMAIL;TYPE=INTERNET:jeffrey@osafoundation.org\r\nFN:Jeffrey Harris\r\nN:Harris;Jeffrey;;;\r\nEND:VCARD\r\n'
>>> j.prettyPrint()
 VCARD
    VERSION: 3.0
    EMAIL: jeffrey@osafoundation.org
    params for  EMAIL:
       TYPE ['INTERNET']
    FN: Jeffrey Harris
    N:  Jeffrey  Harris 
```

#### Parsing vCard objects

```
>>> s = """
... BEGIN:VCARD
... VERSION:3.0
... EMAIL;TYPE=INTERNET:jeffrey@osafoundation.org
... FN:Jeffrey Harris
... N:Harris;Jeffrey;;;
... END:VCARD
... """
>>> v = vobject.readOne( s )
>>> v.prettyPrint()
 VCARD
    VERSION: 3.0
    EMAIL: jeffrey@osafoundation.org
    params for  EMAIL:
       TYPE [u'INTERNET']
    FN: Jeffrey Harris
    N:  Jeffrey  Harris
>>> v.n.value.family
u'Harris'
```


# Release History

### 22 January 2017

**vobject 0.9.4.1** released ([view](https://github.com/eventable/vobject/releases/tag/0.9.4.1)).

To install, use `pip install vobject`, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](https://pypi.python.org/pypi/python-dateutil/) and [six](https://pypi.python.org/pypi/six) are required. Python 2.7 or higher is required.

##### Release Notes

*   Pickling/deepcopy hotfix


***


### 20 January 2017

**vobject 0.9.4** released ([view](https://github.com/eventable/vobject/releases/tag/0.9.4)).

To install, use `pip install vobject`, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](https://pypi.python.org/pypi/python-dateutil/) and [six](https://pypi.python.org/pypi/six) are required. Python 2.7 or higher is required.

##### Release Notes

*   Improved PEP8 compliance
*   Improved Python 3 compatibility
*   Improved encoding/decoding
*   Correct handling of pytz timezones
*   Added tests.py to the PyPi package


***


### 26 August 2016

**vobject 0.9.3** released ([view](https://github.com/eventable/vobject/releases/tag/0.9.3)).

To install, use `pip install vobject`, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](https://pypi.python.org/pypi/python-dateutil/) and [six](https://pypi.python.org/pypi/six) are required. Python 2.7 or higher is required.

##### Release Notes

*   Fixed use of doc in setup.py for -OO mode
*   Added python3 compatibility for base64 encoding
*   Fixed ORG fields with multiple components
*   Handle pytz timezones in iCalendar serialization
*   Use logging instead of printing to stdout


***


### 13 March 2016

**vobject 0.9.2** released ([view](https://github.com/eventable/vobject/releases/tag/0.9.2)).

To install, use `pip install vobject`, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](https://pypi.python.org/pypi/python-dateutil/) and [six](https://pypi.python.org/pypi/six) are required. Python 2.7 or higher is required.

##### Release Notes

*   Better line folding for utf-8 strings
*   Convert unicode to utf-8 to be StringIO compatible


***


### 16 February 2016

**vobject 0.9.1** released ([view](https://github.com/eventable/vobject/releases/tag/0.9.1)).

To install, use `pip install vobject`, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](https://pypi.python.org/pypi/python-dateutil/) and [six](https://pypi.python.org/pypi/six) are required. Python 2.7 or higher is now required.

##### Release Notes

*   Removed lock on dateutil version (>=2.4.0 now works)


***


### 3 February 2016

**vobject 0.9.0** released ([view](https://github.com/eventable/vobject/releases/tag/0.9.0)).

To install, use `pip install vobject`, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil 2.4.0](https://pypi.python.org/pypi/python-dateutil/) and [six](https://pypi.python.org/pypi/six) are required. Python 2.7 or higher is now required.

##### Release Notes

*   Python 3 compatible
*   Updated version of dateutil (2.4.0)
*   More comprehensive unit tests available in tests.py
*   Performance improvements in iteration
*   Test files are included in PyPI download package


***


### 28 January 2016

**vobject 0.8.2** released ([view](https://github.com/eventable/vobject/releases/tag/0.8.2)).

To install, use `pip install vobject`, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Removed unnecessary ez_setup call from setup.py


***


### 27 February 2009

**vobject 0.8.1c** released (SVN revision 217).

To install, use easy_install, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Tweaked change_tz.py to keep it 2.4 compatible


***


### 12 January 2009

**vobject 0.8.1b** released (SVN revision 216).

To install, use easy_install, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Change behavior when import a VCALENDAR or VCARD with an older or absent VERSION line, now the most recent behavior (i.e., VCARD 3.0 and iCalendar, VCALENDAR 2.0) is used


***


### 29 December 2008

**vobject 0.8.0** released (SVN revision 213).

To install, use easy_install, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Changed license to Apache 2.0 from Apache 1.1
*   Fixed a major performance bug in backslash decoding large text bodies
*   Added workaround for strange Apple Address Book parsing of vcard PHOTO, don't wrap PHOTO by default. To disable this behavior, set vobject.vcard.wacky_apple_photo_serialize to False.


***


### 25 July 2008

**vobject 0.7.1** released (SVN revision 208).

To install, use easy_install, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Add change_tz script for converting timezones in iCalendar files


***


### 16 July 2008

**vobject 0.7.0** released (SVN revision 206).

To install, use easy_install, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Allow Outlook's technically illegal use of commas in TZIDs
*   Added introspection help for IPython so tab completion works with vobject's custom __getattr__
*   Made vobjects pickle-able
*   Added tolerance for the escaped semi-colons in RRULEs a Ruby iCalendar library generates
*   Fixed [Bug 12245](https://bugzilla.osafoundation.org/show_bug.cgi?id=12245), setting an rrule from a dateutil instance missed BYMONTHDAY when the number used is negative


***


### 30 May 2008

**vobject 0.6.6** released (SVN revision 201).

To install, use easy_install, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Fixed [bug 12120](https://bugzilla.osafoundation.org/show_bug.cgi?id=12120), unicode TZIDs were failing to parse.


***


### 28 May 2008

**vobject 0.6.5** released (SVN revision 200).

To install, use easy_install, or download the archive and untar, run python setup.py install. Tests can be run via python setup.py test. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Fixed [bug 9814](https://bugzilla.osafoundation.org/show_bug.cgi?id=9814), quoted-printable data wasn't being decoded into unicode, thanks to Ilpo Nyyssönen for the fix.
*   Fixed [bug 12008](https://bugzilla.osafoundation.org/show_bug.cgi?id=12008), silently translate buggy Lotus Notes names with underscores into dashes.


***


### 21 February 2008

**vobject 0.6.0** released (SVN revision 193).

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Added VAVAILABILITY support, thanks to the Calendar Server team.
*   Improved unicode line folding.


***


### 14 January 2008

**vobject 0.5.0** released (SVN revision 189).

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Updated to more recent ez_setup, vobject wasn't successfully installing.


***


### 19 November 2007

**vobject 0.4.9** released (SVN revision 187).

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Tolerate invalid UNTIL values for recurring events
*   Minor improvements to logging and tracebacks
*   Fix serialization of zero-delta durations
*   Treat different tzinfo classes that represent UTC as equal
*   Added ORG behavior to vCard handling, native value for ORG is now a list.


***


### 7 January 2007

**vobject 0.4.8** released (SVN revision 180).

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Fixed problem with the UNTIL time used when creating a dateutil rruleset.


***


### 21 December 2006

**vobject 0.4.7** released (SVN revision 172), hot on the heals of yesterday's 0.4.6.

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Fixed a problem causing DATE valued RDATEs and EXDATEs to be ignored when interpreting recurrence rules
*   And, from the short lived vobject 0.4.6, added an ics_diff module and an ics_diff command line script for comparing similar iCalendar files


***


### 20 December 2006

**vobject 0.4.6** released (SVN revision 171)

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Added an ics_diff module and an ics_diff command line script for comparing similar iCalendar files


***


### 8 December 2006

**vobject 0.4.5** released (SVN revision 168)

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Added ignoreUnreadable flag to readOne and readComponents
*   Tolerate date-time or date fields incorrectly failing to set VALUE=DATE for date values
*   Cause unrecognized lines to default to use a text behavior, so commas, carriage returns, and semi-colons are escaped properly in unrecognized lines


***


### 9 October 2006

**vobject 0.4.4** released (SVN revision 159)

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 1.1 or later is required. Python 2.4 is also required.

##### Release Notes

*   Merged in Apple CalendarServer patches as of CalendarServer-r191
*   Added copy and duplicate code to base module
*   Improved recurring VTODO handling
*   Save TZIDs when parsed and use them as back up TZIDs when serializing


***


### 22 September 2006

**vobject 0.4.3** released (SVN revision 157)

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. Python 2.4 is also required.

##### Release Notes

*   Added support for PyTZ tzinfo classes.


***


### 29 August 2006

**vobject 0.4.2** released (SVN revision 153)

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. Python 2.4 is also required.

##### Release Notes

*   Updated ez_setup.py to use the latest setuptools.


***


### 4 August 2006

**vobject 0.4.1** released (SVN revision 152)

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. Python 2.4 is also required.

##### Release Notes

*   When vobject encounters ascii, it now tries utf-8, then utf-16 with either LE or BE byte orders, searching for BEGIN in the decoded string to determine if it's found an encoding match. readOne and readComponents will no longer work on arbitrary Versit style ascii streams unless the optional findBegin flag is set to False


***


### 2 August 2006

**vobject 0.4.0** released (SVN revision 151)

To install, use easy_install, or download the archive and untar, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. Python 2.4 is also required.

##### Release Notes

*   Workarounds for common invalid files produced by Apple's iCal and AddressBook
*   Added getChildValue convenience method
*   Added experimental hCalendar serialization
*   Handle DATE valued EXDATE and RRULEs better


***


### 17 February 2006

**vobject 0.3.0** released (SVN revision 129)

To install, untar the archive, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. Python 2.4 is also required.

##### Release Notes

*   Changed API for accessing children and parameters, attributes now return the first child or parameter, not a list. See [usage](usage.html) for examples
*   Added support for groups, a vcard feature
*   Added behavior for FREEBUSY lines
*   Worked around problem with dateutil's treatment of experimental properties (bug 4978)
*   Fixed bug 4992, problem with rruleset when addRDate is set


***


### 9 January 2006

**vobject 0.2.3** released (SVN revision 104)

To install, untar the archive, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. Python 2.4 is also required.

##### Release Notes

*   Added VERSION line back into native iCalendar objects
*   Added a first stab at a vcard module, parsing of vCard 3.0 files now gives structured values for N and ADR properties
*   Fix bug in regular expression causing the '^' character to not parse


***


### 4 November 2005

**vobject 0.2.2** released (SVN revision 101)

To install, untar the archive, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. Python 2.4 is also required.

##### Release Notes

*   Fixed problem with add('duration')
*   Fixed serialization of EXDATEs which are dates or have floating timezone
*   Fixed problem serializing timezones with no daylight savings time


***


### 10 October 2005

**vobject 0.2.0** released (SVN revision 97)

To install, untar the archive, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. Python 2.4 is also required.

##### Release Notes

*   Added serialization of arbitrary tzinfo classes as VTIMEZONEs
*   Removed unused methods
*   Changed getLogicalLines to use regular expressions, dramatically speeding it up
*   Changed rruleset behavior to use a property for rruleset


***


### 30 September 2005

**vobject 0.1.4** released (SVN revision 93)

To install, untar the archive, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. As of this release, Python 2.4 is also required.

##### Release Notes

*   Changed parseLine to use regular expression instead of a state machine, reducing parse time dramatically


***


### 1 July 2005

**vobject 0.1.3** released (SVN revision 88)

To install, untar the archive, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) 0.9 or later is required. As of this release, Python 2.4 is also required.

##### Release Notes

*   Added license and acknowledgements.
*   Fixed the fact that defaultSerialize wasn't escaping linefeeds
*   Updated backslashEscape to encode CRLF's and bare CR's as linefeeds, which seems to be what RFC2445 requires


***


### 24 March 2005

**vobject 0.1.2** released (SVN revision 83)

To install, untar the archive, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) is required. You'll need to apply this [patch](dateutil-0.5-tzoffset-bug.patch) to be able to read certain VTIMEZONEs exported by Apple iCal, or if you happen to be in Europe!

patch -R $PYTHONLIB/site-packages/dateutil/tz.py dateutil-0.5-tzoffset-bug.patch

##### Release Notes

*   Fixed printing of non-ascii unicode.
*   Fixed bug preventing content lines with empty contents from parsing.


***


### 25 January 2005

**vobject 0.1.1** released (SVN revision 82)

To install, untar the archive, run python setup.py install. [dateutil](http://labix.org/python-dateutil#head-2f49784d6b27bae60cde1cff6a535663cf87497b) is required. You'll need to apply this [patch](dateutil-0.5-tzoffset-bug.patch) to be able to read certain VTIMEZONEs exported by Apple iCal, or if you happen to be in Europe!

patch -R $PYTHONLIB/site-packages/dateutil/tz.py dateutil-0.5-tzoffset-bug.patch

##### Release Notes

*   Various bug fixes involving recurrence.
*   TRIGGER and VALARM behaviors set up.


***


### 13 December 2004

**vobject 0.1** released (SVN revision 70)

##### Release Notes

*   Parsing all iCalendar files should be working, please [file a bug](bugs/) if you can't read one!
*   Timezones can be set for datetimes, but currently they'll be converted to UTC for serializing, because VTIMEZONE serialization isn't yet working.
*   RRULEs can be parsed, but when they're serialized, they'll be converted to a maximum of 500 RDATEs, because RRULE serialization isn't yet working.
*   To parse unicode, see [issue 4](http://vobject.skyhouseconsulting.com/bugs/issue4).
*   Much more testing is needed, of course!
