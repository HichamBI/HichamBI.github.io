---
layout: post
title:  "Export SVG to PDF with Batik SVG Toolkit"
date:   2015-03-15 16:09:00
img: "posts/svg-to-pdf-export.svg"
---

In this post, (The First !) i will briefly show you how to create a PDF report including text and SVG image using Batik SVG Toolkit. 

#### Why Batik SVG Toolkit ?

There is several technologies in java generating PDF documents and able to fill textual and graphical data.

Examples of technologies : 

+ iText: Very useful library, but you must buy the commercial licence if you want to use it without disclosing the source code of your own applications.

+ PDFBox: Under Apache License v2.0, it seems to be the best alternative to iText, but didn't support SVG images.

+ Flying Saucer: I didn't check this library yet, it use iText to generate PDF files. 

Note that all these libraries use Batik to manipulate SVG files.
 
#### How ?

THe main idea is to use Batik tools to create a main SVG stream that contains all our text, images ... and trancode it and generate a PDF file.

#####- Dependencies :

We need to import this dependencies to our project : 

{% highlight html %}
<dependency>
 <groupId>org.apache.xmlgraphics</groupId>
 <artifactId>fop</artifactId>
 <version>1.1</version>
 <exclusions>
  <exclusion>
   <artifactId>avalon-framework-api</artifactId>
   <groupId>org.apache.avalon.framework</groupId>
  </exclusion>
  <exclusion>
   <artifactId>avalon-framework-impl</artifactId>
   <groupId>org.apache.avalon.framework</groupId>
  </exclusion>
 </exclusions>
</dependency>
<!-- these two are to correct issues in fop dependency -->
<dependency>
 <groupId>avalon-framework</groupId>
 <artifactId>avalon-framework-api</artifactId>
 <version>4.2.0</version>
</dependency>
<dependency>
 <groupId>avalon-framework</groupId>
 <artifactId>avalon-framework-impl</artifactId>
 <version>4.2.0</version>
</dependency>
<dependency>
 <groupId>org.apache.xmlgraphics</groupId>
 <artifactId>batik-codec</artifactId>
 <version>1.7</version>
</dependency>
{% endhighlight %}

##### Creating the main class :

We are going to create a class that contains all methods we need to create our report:

#####- Class initialization
Create SVGGraphics2D instance that will allow us to generate the main SVG stream.

1.  Get a DOMImplementation and create an instance of org.w3c.dom.Document.
2.  Get a SVGGeneratorContext from Document instance and create an instance of SVGGraphics2D.
3.  Get an XMLParser and create an instance of SAXSVGDocumentFactory.
4.  Create a BridgeContext.
5.  Create a GVTBuilder.

{% highlight ruby %}

private void initialize() {
    DOMImplementation domImpl = GenericDOMImplementation.getDOMImplementation();
    String svgNamespaceURI = "http://www.w3.org/2000/svg";
    Document document = domImpl.createDocument(svgNamespaceURI, "svg", null);
    
    SVGGeneratorContext sVGGeneratorContext = SVGGeneratorContext.createDefault(document);
    svgGraphics2D = new SVGGraphics2D(sVGGeneratorContext, false);
    
    String parser = XMLResourceDescriptor.getXMLParserClassName();
    factory = new SAXSVGDocumentFactory(parser);
    
    UserAgent userAgent = new UserAgentAdapter();
    DocumentLoader loader = new DocumentLoader(userAgent);
    ctx = new BridgeContext(userAgent, loader);
    ctx.setDynamicState(BridgeContext.DYNAMIC);
    
    builder = new GVTBuilder();
}

{% endhighlight %}

#####- addSvgImage(File svgFile, float xPosition, float yPosition)
Create GraphicsNode from svgFile and put it in our SVGGraphics2D instance.

1. Get an SVGDocument from svgFile and create a GraphicsNode using BridgeContext and GVTBuilder.
2. Use AffineTransform to put the svgFile to the good position and paint the GraphicsNode.
{% highlight ruby %}  
    public void addSvgImage(File svgFile, float xPosition, float yPosition) {
    SVGDocument svgDocument = factory.createSVGDocument(svgFile.toURL().toString());
    GraphicsNode mapGraphics = builder.build(ctx, svgDocument);
    
    AffineTransform transformer = new AffineTransform();
    transformer.translate(xPosition, yPosition);
    
    mapGraphics.setTransform(transformer);
    mapGraphics.paint(svgGraphics2D);
}
{% endhighlight %}

#####- addTextLine(String text, Font font, float xPosition, float yPosition)
We are going to put our text in the SVGGraphics2D

{% highlight ruby %}  
  public void addTextLine(String text, Font font, float xPosition, float yPosition) {
   svgGraphics2D.setFont(font);
   svgGraphics2D.drawString(text, xPosition, yPosition);
  }
{% endhighlight %} 

#####- getPDFStream()
Generate the main SVG stream and transcode it to an PDF stream.

1. Get svgGraphics2D stream.
2. Generate a PDF stream using PDFTranscoder.

{% highlight ruby %}  
  public ByteArrayOutputStream getPDFStream() throws IOException, TranscoderException {
   ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
   Writer out = new OutputStreamWriter(outputStream, "UTF-8");
   svgGraphics2D.stream(out, true);
   String svg = new String(outputStream.toByteArray(), "UTF-8");
   return transcode(svg);
  }
  
  private ByteArrayOutputStream transcode(String svg) throws TranscoderException {
   TranscoderInput input = new TranscoderInput(new StringReader(svg));
   ByteArrayOutputStream output = new ByteArrayOutputStream();
   TranscoderOutput transOutput = new TranscoderOutput(output);
   SVGAbstractTranscoder transcoder = new PDFTranscoder();
   
   transcoder.addTranscodingHint(SVGAbstractTranscoder.KEY_WIDTH, documentWidth);
   transcoder.addTranscodingHint(SVGAbstractTranscoder.KEY_HEIGHT, documentHeight);
   
   transcoder.transcode(input, transOutput);
   
  return output;
}
{% endhighlight %}

#####- Testing :
{% highlight ruby %}  
  public static void main(String... arg) {
   PDFDoc pdfDoc = new PDFDoc(600f, 588f);
   
   Font fontBold = new Font("Arial", Font.BOLD, 13);
   Font fontNormal = new Font("Arial", Font.PLAIN, 13);
   
   pdfDoc.addTextLine("Entity", fontBold, 15, 40);
   pdfDoc.addTextLine("Benchmark : MCI Euro", fontNormal, 15, 53);
   pdfDoc.addTextLine("Approval Date : 15/09/2015", fontNormal, 15, 66);
   pdfDoc.addTextLine("Other Parameters", fontBold, 15, 92);
   pdfDoc.addTextLine("Benchmark Type : Master Configuration", fontNormal, 15, 105);
   pdfDoc.addTextLine("Currency : EUR", fontNormal, 15, 118);
   
   File file = new File("gradient.svg");
   pdfDoc.addSvgImage(file, 0, 131);
   
   ByteArrayOutputStream pdfPageStream = pdfDoc.getPDFStream();
   File temp = new File("result.pdf");
   
   try(FileOutputStream fileOut = new FileOutputStream(temp)) {
    fileOut.write(pdfPageStream.toByteArray());
   }
  }
{% endhighlight %} 

##### What about SVG images with gradient paint ?
If you try tu use this code with SVG that include gradient pain, all gradients will be turned to black color.
To fix this, we must define an ExtensionHandler and add it to our SVGGeneratorContext.

#####-   The GradientExtensionHandler class
We are going to extend the DefaultExtensionHandler and override the handlePaint methode.

1. For each gradient type, we are going to define an SVGPaintDescriptor.
2. For each SVGPaintDescriptor, we are going to define a gradient node element and set several attributes like color starts and stops, FX, FY etc.
3. Add GradientExtensionHandler to our SVGGeneratorContext.

{% highlight ruby %} 
   class GradientExtensionHandler extends DefaultExtensionHandler {
    @Override
    public SVGPaintDescriptor handlePaint(Paint paint, SVGGeneratorContext generatorCtx) {
     if (paint instanceof LinearGradientPaint)
      return getLinearGradientPaintDescriptor((LinearGradientPaint) paint, generatorCtx);
     else if (paint instanceof RadialGradientPaint) {
      return getRadialGradientPaintDescriptor((RadialGradientPaint) paint, generatorCtx);
     }
      return super.handlePaint(paint, generatorCtx);
    }
    
    // You find the complete class implementation in my github repository. 
   }
{% endhighlight %} 

Finally, set GradientExtensionHandler to SVGGeneratorContext :  

{% highlight ruby %} 
    sVGGeneratorContext.setExtensionHandler(new GradientExtensionHandler());
{% endhighlight %}

That's all ! You can find code source [here](http://github.com/HichamBI/svg-pdf-exporter).
