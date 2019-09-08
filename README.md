# svgliteForAffinity

## Export svg plots in R for Affinity Designer

Scalable Vector Graphics (SVG) is an XML-based vector image format for two-dimensional graphics with support for interactivity and animation. I find it convenient to work with since details in plots and figures are not lost when zooming in. 

Editing SVG files can be done through vector graphics editors such as Adobe Illustrator or Affinity Designer, the latter being free of charge for anyone working at Chalmers. Plots and figures generated in R can be saved in SVG format using the ggsave function from the ggplot package.

However, I noticed that figures saved through ggsave the ggsave are loaded incorrectly in Affinity Designer due to inproper formatting (more details further down).

__For example, the following plot:__

![alt text](https://raw.githubusercontent.com/angelolimeta/svgliteForAffinity/master/Correct.png)

__Looks like this when opened in Affinity Designer:__

![alt text](https://raw.githubusercontent.com/angelolimeta/svgliteForAffinity/master/Incorrect.png)

## How to fix:

First, uninstall the svglite package from R if you already have it installed.
Run this command in R studio:
```r
remove.packages("svglite", lib="~/Library/R/3.6/library")
```

Next, download the file "svglite_1.2.2.tar.gz" from this repo. Either click on the file and hit download or use wget from a terminal:

```bash
wget https://github.com/angelolimeta/svgliteForAffinity/raw/master/svglite_1.2.2.tar.gz
```

Finally, install the package in R studio with the install.packages command:

```r
install.packages("/path/to/svglite_1.2.2.tar.gz", repos = NULL)
```
(Note that you need to replace /path/to/ with the path to the downloaded file)

__Now you can save correctly formatted plots for use with Affinity Designer!__

## Changes made to the svglite package

Apparently Affinity Designer's SVG parser doesn't handle the formating for multiple CSS style tags this way:
```xml
<?xml version='1.0' encoding='UTF-8' ?>
<svg xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' viewBox='0 0 216.00 270.00'>
<defs>
  <style type='text/css'><![CDATA[
    line, polyline, path, rect, circle {
      fill: none;
      stroke: #000000;
      stroke-linecap: round;
      stroke-linejoin: round;
      stroke-miterlimit: 10.00;
    }
  ]]></style>
</defs>
```

Multiple CSS style tagers need to split up into individual nodes as follows:
```xml
...
<defs>
  <style>
    line {
      fill: none;
      stroke: #000000;
      stroke-linecap: round;
      stroke-linejoin: round;
      stroke-miterlimit: 10.00;
    }
    polyline {
      fill: none;
      stroke: #000000;
      stroke-linecap: round;
      stroke-linejoin: round;
      stroke-miterlimit: 10.00;
    }
    path {
      fill: none;
      stroke: #000000;
      stroke-linecap: round;
      stroke-linejoin: round;
      stroke-miterlimit: 10.00;
    }
    ...
  </style>
</defs>
...
```

Affinity designer does also not treat CDATA sections:
```xml
<style type='text/css'><![CDATA[
    ...
  ]]></style>
```

Taking a quick look into the ggplot source package (ggplot2_2.2.1.tar.gz) shows that they use there in the "save.r" file a function from another package ...

```r
...   
svg =  function(...) svglite::svglite(...),
...
```

Looking in turn into the sources of that svglite package (svglite_1.2.1.tar.gz) shows that they use/call C++ code there, see the file "devSVG.cpp". That C++ source file declares a class "SVGDesc" which has a method "void svg_new_page(const pGEcontext gc, pDevDesc dd)" ...

```cpp
void svg_new_page(const pGEcontext gc, pDevDesc dd) {
BEGIN_RCPP

  SVGDesc *svgd = (SVGDesc*) dd->deviceSpecific;
  SvgStreamPtr stream = svgd->stream;

  if (svgd->pageno > 0) {
    Rcpp::stop("svglite only supports one page");
  }

  if (svgd->standalone)
    (*stream) << "<?xml version='1.0' encoding='UTF-8' ?>\n";

  (*stream) << "<svg";
  if (svgd->standalone){
    (*stream) << " xmlns='http://www.w3.org/2000/svg'";
    //http://www.w3.org/wiki/SVG_Links
    (*stream) << " xmlns:xlink='http://www.w3.org/1999/xlink'";
  }

  (*stream) << " viewBox='0 0 " << dd->right << ' ' << dd->bottom << "'>\n";

  // Initialise clipping the same way R does
  svgd->clipx0 = 0;
  svgd->clipy0 = dd->bottom;
  svgd->clipx1 = dd->right;
  svgd->clipy1 = 0;

  // Setting default styles
  (*stream) << "<defs>\n";
  (*stream) << "  <style type='text/css'><![CDATA[\n";
  (*stream) << "    line, polyline, path, rect, circle {\n";
  (*stream) << "      fill: none;\n";
  (*stream) << "      stroke: #000000;\n";
  (*stream) << "      stroke-linecap: round;\n";
  (*stream) << "      stroke-linejoin: round;\n";
  (*stream) << "      stroke-miterlimit: 10.00;\n";
  (*stream) << "    }\n";
  (*stream) << "  ]]></style>\n";
  (*stream) << "</defs>\n";

  (*stream) << "<rect width='100%' height='100%'";
  write_style_begin(stream);
  write_style_str(stream, "stroke", "none", true);
  if (is_filled(gc->fill))
    write_style_col(stream, "fill", gc->fill);
  else
    write_style_col(stream, "fill", dd->startfill);
  write_style_end(stream);
  (*stream) << "/>\n";

  svgd->stream->flush();
  svgd->pageno++;

VOID_END_RCPP
}
```

Let's rewrite that above shown method and change the stream outputs accordingly ...
```cpp
...
// Setting default styles
  (*stream) << "<defs>\n";
  (*stream) << "  <style>\n";
  (*stream) << "    line {\n";
  (*stream) << "      fill: none;\n";
  (*stream) << "      stroke: #000000;\n";
  (*stream) << "      stroke-linecap: round;\n";
  (*stream) << "      stroke-linejoin: round;\n";
  (*stream) << "      stroke-miterlimit: 10.00;\n";
  (*stream) << "    }\n";
  (*stream) << "    polyline {\n";
  (*stream) << "      fill: none;\n";
  (*stream) << "      stroke: #000000;\n";
  (*stream) << "      stroke-linecap: round;\n";
  (*stream) << "      stroke-linejoin: round;\n";
  (*stream) << "      stroke-miterlimit: 10.00;\n";
  (*stream) << "    }\n";
  (*stream) << "    path {\n";
  (*stream) << "      fill: none;\n";
  (*stream) << "      stroke: #000000;\n";
  (*stream) << "      stroke-linecap: round;\n";
  (*stream) << "      stroke-linejoin: round;\n";
  (*stream) << "      stroke-miterlimit: 10.00;\n";
  (*stream) << "    }\n";
  (*stream) << "    rect {\n";
  (*stream) << "      fill: none;\n";
  (*stream) << "      stroke: #000000;\n";
  (*stream) << "      stroke-linecap: round;\n";
  (*stream) << "      stroke-linejoin: round;\n";
  (*stream) << "      stroke-miterlimit: 10.00;\n";
  (*stream) << "    }\n";
  (*stream) << "    circle {\n";
  (*stream) << "      fill: none;\n";
  (*stream) << "      stroke: #000000;\n";
  (*stream) << "      stroke-linecap: round;\n";
  (*stream) << "      stroke-linejoin: round;\n";
  (*stream) << "      stroke-miterlimit: 10.00;\n";
  (*stream) << "    }\n";
  (*stream) << "  </style>\n";
  (*stream) << "</defs>\n";
...
```

Download and extract the source:
```bash
wget https://cran.r-project.org/src/contrib/svglite_1.2.2.tar.gz
tar -xvzf svglite_1.2.2.tar.gz
```

This results in a direcotry named svglite.
Let's open this folder and navigate to src/devSVG.cpp and make the change mentioned above.
Now let's navigate back to the directory where you downloaded the package and build our modified package:
```bash
R CMD build svglite
```

This will result in a file named svglite_1.2.2.tar.gz
Lastly, install the modified archive:
```
R CMD INSTALL svglite_1.2.2.tar.gz
```

__Now you can use this version of the svglite package together with ggplot in R and have your figures load correctly in Affinity Desinger!__

