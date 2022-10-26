---
title: 'Getting data from PDFs with JRuby'
date: 06-10-2014
permalink: /posts/2014/10/getting-data-from-pdfs-with-jruby/
redirect_from:
  - /getting-data-from-pdfs-with-jruby
tags:
  - ruby
  - archive
---

 There are many solutions for getting data from pdfs. I'm going to describe how to use the excellent Java library PDFTextStream by Chas Emerick (of Clojure fame) to get data out of tricky pdfs.

 ## Why PDFTextStream?

 Quite simply, it's the best PDF extraction library I've come across in terms of features and performance. It handles layouts and formatting very well and the xml output gives some useful tags for data extraction.

 ### Getting the library

 Head over to [http://snowtide.com/downloads](http://snowtide.com/downloads) and download the latest Java version (2.7.0 at the time of writing) and unzip into a folder called `jruby-demo`.

 ### Some JRuby/Java interop

 Create a file at `jruby-demo/pdf-extractor.rb` with the following contents:

 ```ruby
 require 'java'
 require 'json'
 require 'PDFTextStream.Java-2.7.0/lib/PDFTextStream.jar'
 $CLASSPATH << 'PDFTextStream.Java-2.7.0/src'

 java_import com.snowtide.pdf.PDFTextStream
 java_import com.snowtide.pdf.OutputTarget # To output plain text
 java_import "pdfts.examples.XMLOutputTarget" # To output XML
 java_import java.lang.StringBuilder

 pdf_file_path = File.join(Dir.pwd, ARGV[0])

   sb = StringBuilder.new # Requires Java StringBuilder for some reason
   pdfts = Java::ComSnowtidePdf::PDFTextStream.new(pdf_file_path)

   case ARGV[1]
     when "xml"
       # XMLOutputTarget keeps the formatting tags of
       # the input PDF - useful if the source uses bold or italics etc.
       ot = XMLOutputTarget.new

       pdfts.pipe(ot)
       pdfts.close
       puts ot.getXMLAsString
     when "standard"
       # Normal OutputTarget reformats the text to handle
       # column layouts
       ot = Java::ComSnowtidePdf::OutputTarget.new(sb)

       pdfts.pipe(ot)
       pdfts.close
       puts sb.to_s
     when "visual"
       # VisualOutputTarget is better at preserving layout in
       # the conversion to text e.g. tables
       ot = Java::ComSnowtidePdf::VisualOutputTarget.new(sb)

       pdfts.pipe(ot)
       pdfts.close
       puts sb.to_s
     else
       # VisualOutputTarget is better at preserving layout in
       # the conversion to text e.g. tables
       ot = Java::ComSnowtidePdf::VisualOutputTarget.new(sb)

       pdfts.pipe(ot)
       pdfts.close
       puts sb.to_s
   end

 ```

 ### Extracting some text

 Move a pdf into the folder, install jruby and then run:

 ```bash
 cd jruby-demo
 jruby pdf-extractor.rb name-of-your-pdf.pdf standard
 ```

 and after a few seconds of jvm warm up time you should start to see text on STDOUT.


 ### Different extraction modes

 `standard` - This handles column layouts (common in pdfs) and reflows them to make sure the text reads in the correct order.

 `visual` - This preserves the text spacing on the page which is useful for tabular data.

 `xml` - If the source data has bold or italic text, this processor outputs xml markup which can be useful for further processing with Nokogiri or other similar libraries.

