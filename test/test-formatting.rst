.. contents::
.. sectnum::
.. role:: raw-html(raw)
    :format: html

==============
Test Heading 1
==============

Paragraph.

Test Heading 2
==============

Paragraph.

Test Heading 3
--------------

Paragraph.

Test Heading 4
--------------

Paragraph.

==============
Line breaks
==============

Forcing a line break in text works as follows:

First line. :raw-html:`<br />`
Second line. :raw-html:`<br />`
Third line.

The line block syntax should NOT be used since it only partly works,
in that it renders correctly when viewing the actual
file but renders incorrectly in diffs:

| First line.
| Second line.
| Third line.

==============
Glossary
==============

Field list
==============

:what: Field lists map field names to field bodies, like
       database records.  They are often part of an extension
       syntax.

:how: The field marker is a colon, the field name, and a
      colon.

      The field body may contain one or more body elements,
      indented relative to the field marker.

Definition list
===============

what
    Definition lists associate a term with a definition.

how
    The term is a one-line phrase, and the definition is one
    or more paragraphs or body elements, indented relative to
    the term.

==============
Indentation
==============

Paragraph:

::

    Literal block

Paragraph: ::

    Literal block

Paragraph::

    Literal block

This is a top-level paragraph.

    This paragraph belongs to a first-level block quote.

    Paragraph 2 of the first-level block quote.

This is a typical paragraph.  An indented literal block follows.

::

    for a in [5,4,3,2,1]:   # this is program code, shown as-is
        print a
    print "it's..."
    # a literal block continues until the indentation ends

This text has returned to the indentation of the first paragraph,
is outside of the literal block, and is therefore treated as an
ordinary paragraph.

Another indented code block follows. ::

    for a in [5,4,3,2,1]:   # this is program code, shown as-is
        print a
    print "it's..."
    # a literal block continues until the indentation ends

==============
Notes
==============

Notes and warnings do NOT render as such in github at all, as seen below:

.. note::
   This is note text. Use a note for information you want the user to
   pay particular attention to.

   If note text runs over a line, make sure the lines wrap and are indented to
   the same level as the note tag. If formatting is incorrect, part of the note
   might not render in the HTML output.

   Notes can have more than one paragraph. Successive paragraphs must
   indent to the same level as the rest of the note.

.. warning::
    This is warning text. Use a warning for information the user must
    understand to avoid negative consequences.

    Warnings are formatted in the same way as notes. In the same way,
    lines must be broken and indented under the warning tag.

==============
Tables
==============

Simple tables
==============

Here's a simple table:

  ===========  ================  ===========================
  Row          Value             Notes       
  ===========  ================  ===========================
  1            Value 1           Note 1
  2            Value 2           Note 2
  3            Value 3           Note 3
  ===========  ================  ===========================

Here's a simple table with multiple source lines in the same cell:

  ===========  ==================  ===========================
  Row          Value               Notes       
  ===========  ==================  ===========================
  1            Value 1             Note 1
  2            Here's a row        Note 2
               with multiple
               lines in the
               source.
  3            Value 3             Note 3
  ===========  ==================  ===========================

Here's a simple table that tries to render multiple lines in the same cell:

  ===========  ==================  ===========================
  Row          Value               Notes       
  ===========  ==================  ===========================
  1            Value 1             Note 1
  2            Here's a row        Note 2
               with multiple
               :raw-html:`<br />`
               lines in the
               output.
  3            Value 3             Note 3
  ===========  ==================  ===========================

Here's a simple table that tries to render multiple lines of pseudocode in the same cell:

  ===========  ==================  ===========================
  Row          Value               Notes       
  ===========  ==================  ===========================
  1            Value 1             Note 1
  2            ``if foo``          Note 2
               :raw-html:`<br />`
               |``indent``
               :raw-html:`<br />`
               ``done``
  3            Here's a row with   Note 3
               :raw-html:`<br />`
               | indented
               :raw-html:`<br />`
               text.
  ===========  ==================  ===========================

Here's a simple table that tries to render multiple lines of code in the same cell:

  ===========  ==================  ===========================
  Row          Value               Notes       
  ===========  ==================  ===========================
  1            Value 1             Note 1
  2            ``if foo``          Note 2
               ::

                   ``indented``

               :raw-html:`<br />`
               ``done``
  3            Here's a row with   Note 3
               :raw-html:`<br />`
               | indented
               :raw-html:`<br />`
               text.
  ===========  ==================  ===========================

Grid tables
==============

Here's a grid table:

+------------+-----------------+---------------------------+
| Row        | Value           | Notes                     |
+============+=================+===========================+
| 1          | Value 1         | Note 1                    |
+------------+-----------------+---------------------------+
| 2          | Value 2         | Note 2                    |
+------------+-----------------+---------------------------+
| 3          | Value 3         | Note 3                    |
+------------+-----------------+---------------------------+

Here's a grid table with multiple source lines in the same cell:

+------------+-----------------+---------------------------+
| Row        | Value           | Notes                     |
+============+=================+===========================+
| 1          | Value 1         | Note 1                    |
+------------+-----------------+---------------------------+
| 2          | This text spans | Note 2                    |
|            | multiple lines  |                           |
|            | in the source.  |                           |
+------------+-----------------+---------------------------+
| 3          | Value 3         | Note 3                    |
+------------+-----------------+---------------------------+

Here's a grid table that tries to render multiple lines in the same cell:

+------------+--------------------+---------------------------+
| Row        | Value              | Notes                     |
+============+====================+===========================+
| 1          | Value 1            | Note 1                    |
+------------+--------------------+---------------------------+
| 2          | This text spans    | Note 2                    |
|            | multiple lines     |                           |
|            | :raw-html:`<br />` |                           |
|            | in the output.     |                           |
+------------+--------------------+---------------------------+
| 3          | Value 3            | Note 3                    |
+------------+--------------------+---------------------------+
