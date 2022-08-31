.. contents::
.. sectnum::
.. role:: raw-html(raw)
    :format: html

==============
Test Heading 1
==============

Forcing a line break in text can work as follows:

First line. :raw-html:`<br />`
Second line. :raw-html:`<br />`
Third line.

But line block syntax does not seem to work:

| First line.
| Second line.
| Third line.

.. note::

       This is a note.

Test Heading 2
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

Test Heading 3
--------------

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


Test Heading 4.
~~~~~~~~~~~~~~~

Paragraph.

