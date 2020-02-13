ansi art display in linux terminal
==================================

Yes! It is posssible to display good old 90s ANSI art artwork in your shinny modern linux terminal..

The easy option: convert old DOS codepage 437 to UTF-8
------------------------------------------------------

	iconv -f "CP437" -t "UTF-8" original.ans -o converted.ans
	cat converted.ans

be sure to be at the right size:

	xfce4-terminal --geometry 80x40

	