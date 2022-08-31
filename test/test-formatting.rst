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

Forcing a line break in text can work as follows:

First line. :raw-html:`<br />`
Second line. :raw-html:`<br />`
Third line.

The line block syntax only partly works, in that it renders correctly when viewing the actual
file but renders incorrectly in diffs:

| First line.
| Second line.
| Third line.

==============
Notes
==============

Notes and warnings do not render as such in github at all, as seen below:

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

First syntax
==============

Here's a plain table in one syntax:

  ===========  ================  ===========================
  Row          Value             Notes       
  ===========  ================  ===========================
  1            Value 1           Note 1
  2            Value 2           Note 2
  3            Value 3           Note 3
  ===========  ================  ===========================

Here's a table with multiple source lines in the same cell:

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

Here's a table that tries to render multiple lines in the same cell:

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

Here's a table that tries to render multiple lines of code in the same cell:

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
               |indented
               :raw-html:`<br />`
               text.
  ===========  ==================  ===========================


Second Syntax
==============

Here's a plain table in a second syntax:

+------------+-----------------+---------------------------+
| Row        | Value           | Notes                     |
+============+=================+===========================+
| 1          | Value 1         | Note 1                    |
+------------+-----------------+---------------------------+
| 2          | Value 2         | Note 2                    |
+------------+-----------------+---------------------------+
| 3          | Value 3         | Note 3                    |
+------------+-----------------+---------------------------+

Here's a table with multiple source lines in the same cell:

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

Here's a table that tries to render multiple lines in the same cell:

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
