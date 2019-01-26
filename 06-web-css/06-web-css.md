---
title: "Webscraping: (1) Server-side and CSS"
author:
  name: Grant R. McDermott | University of Oregon
  # affiliation: EC 607
  # email: grantmcd@uoregon.edu
date: EC 607 | Lecture 6  #"26 January 2019"
output: 
  html_document:
    theme: flatly
    highlight: haddock 
    # code_folding: show
    toc: yes
    toc_depth: 4
    toc_float: yes
    keep_md: true
---



## Software requirements

### External software

Today we'll be using [SelectorGadget](https://selectorgadget.com/), which is a Chrome extension that makes it easy to discover CSS selectors.  (Install the extension directly [here](https://chrome.google.com/webstore/detail/selectorgadget/mhjhnkcfbdhnjickkkdbjoemdmbfginb).) Please note that SelectorGadget is only available for Chrome. If you prefer using Firefox, then you can try [ScrapeMate](https://addons.mozilla.org/en-US/firefox/addon/scrapemate/).

### R packages 

- **New:** `rvest`, `janitor`
- **Already used:** `tidyverse`, `lubridate`, `hrbrthemes`

Recall that `rvest` was automatically installed with the rest of the tidyverse. So you only need to install the small `janitor` package:


```r
## Not run. (Run this manually yourself if you haven't installed the package yet.)
install.packages("janitor")
```

## Server-side Vs. Client-side

The next two lectures are about getting data, or "content", off the web and onto our computers. We're all used to seeing this content in our browers (Chrome, Firefox, etc.). So we know that it must exist somewhere. However, it's important to realise that there are actually two ways that web content gets rendered in a browser: 

1. Server-side
2. Client side

You can read [here](https://www.codeconquest.com/website/client-side-vs-server-side/) for more details (including example scripts), but for our purposes the essential features are as follows: 

### 1. Server-side
- The scripts that "build" the website are run not on our computer, but rather on a host server that sends down all of the HTML code.
- In other words, the information that we see in our browser has already been processed by the host server. 
  - E.g. A table on Wikipedia is already populated with all of the information --- numbers, strings, etc. --- that we see in our browser.
- You can think of this information being embeded directly in the webpage's HTML.
- **Webscraping challenges:** Finding the correct CSS (or Xpath) "selectors". Iterating through dynamic webpages (e.g. "Next page" and "Show More" tabs).
- **Key concepts:** CSS, Xpath, HTML
  
### 2. Client-side
- The website contains an empty template of HTML and CSS. E.g. It might be a "skeleton" table without any values.
- However, when we actually visit the page URL, our browser sends a *request* to the host server.
  - The server then decides whether our request is valid among other things (e.g. no "404"). 
- If everything is okay, then the server sends a *response* script, which our browser executes and uses to populate the HTML template with the specific information that we want.
- **Webscraping challenges:** Finding the API "endpoints" can be tricky, since these are sometimes hidden from view.
- **Key concepts:** APIs, API endpoints

Over the next week, we'll use these lecture notes --- plus some student presentations --- to go over the main differences between the two approaches and cover the implications for any webscraping activity. I want to forewarn you that webscraping typically involves a fair bit of detective work. You will often have to adjust your steps according to the type of data you want, and the steps that worked on one website may not work on another. (Or even work on the same website a few months later). All this is to say that *webscraping involves as much art as it does science*.

The good news is that both server-side and client-side websites allow for webscraping.^[As we'll see during the next lecture, scraping a website or application that is built on a client-side (i.e. API) framework is often easier; particularly when it comes to downloading information *en masse*.] If you can see it in your browser, you can scrape it. 

### Caveat: Ethical and legal limitations

The previous sentence elides some important ethical and legal considerations. Just because you *can* scrape it, doesn't mean you *should*. It is ultimately your responsibility to determine whether a website maintains legal restrictions on the content that it provides. Similarly, the tools that we'll be using are very powerful. It's fairly easy to write up a function or program that can overwhelm a host server or application through the sheer weight of requests. A computer can process commands much, much faster than we can ever type them up manually. We'll come back to the "be nice" motif in the next lecture. 

I won't cover it here, but the [polite package](https://github.com/dmi3kno/polite) provides some helpful tools to maintain web etiquette. It also plays very nicely with the packages that we're going to cover today, so please take a look after you've worked through this lecture.

## Webscraping with `rvest` (server-side)

The primary R package that we'll be using today is Hadley Wickham's [rvest](https://github.com/hadley/rvest). Let's load it now.


```r
library(rvest)
```

`rvest` is a simple webscraping package inspired by Python's [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/), but with extra tidyverse functionality. It is also designed to work with webpages that are built server-side and thus requires knowledge of the relevant CSS selectors... Which means that now is probably a good time for us to cover what these are.

### Student presentation: CSS and SelectorGadget

Time for a student presentation on [CSS](https://developer.mozilla.org/en-US/docs/Learn/CSS/Introduction_to_CSS/How_CSS_works) (i.e Cascading Style Sheets) and [SelectorGadget](http://selectorgadget.com/). Click on the links if you are reading this after the fact. In short, CSS is a language for specifying the appearance of HTML documents (including web pages). It does this by providing web browsers a set of display rules, which are formed by:

1. _Properties._ CSS properties are the "how" of the display rules. These are things like which font family, styles and colours to use, page width, etc.
2. _Selectors._ CSS selectors are the "what" of the display rules. They identify which rules should be applied to which elements. E.g. Text elements that are selected as ".h1" (i.e. top line headers) are usually larger and displayed more prominently than text elements selected as ".h2" (i.e. sub-headers).

The key point is that if you can identify the CSS selector(s) of the content you want, then you can isolate it from the rest of the webpage content that you don't want. This where SelectorGadget comes in. We'll work through an extended example (with a twist!) below, but I highly recommend looking over this [quick vignette](https://cran.r-project.org/web/packages/rvest/vignettes/selectorgadget.html) from Hadley before proceding.

### Application: Mens 100 meters world record progression (Wikipedia)

Okay, let's get to an application. Say that we want to scrape the Wikipedia page on the [Men's 100 metres world record progression](http://en.wikipedia.org/wiki/Men%27s_100_metres_world_record_progression). Open up this page in your browser first. Take a look at its structure: What type of objects does it contain? How many tables does it have? Etc.

Once you've familised yourself with the structure, read in the whole webpage into R using the `rvest::read_html()` function.


```r
wp <- "http://en.wikipedia.org/wiki/Men%27s_100_metres_world_record_progression"
m100 <- 
  wp %>%
  read_html() 
m100
```

```
## {xml_document}
## <html class="client-nojs" lang="en" dir="ltr">
## [1] <head>\n<meta http-equiv="Content-Type" content="text/html; charset= ...
## [2] <body class="mediawiki ltr sitedir-ltr mw-hide-empty-elt ns-0 ns-sub ...
```

As you can see, this is an [XML](https://en.wikipedia.org/wiki/XML) document^[XML stands for Extensible Markup Language and is one of the primary languages used for encoding and formatting web pages.] that contains *everything* needed to render the Wikipedia page. It's kind of like viewing someone's entire LaTeX document (preamble, syntax, etc.) when all we want are the data from some tables in their paper.

#### Table 1: Pre-IAAF (1881--1912)

Now, let's try to isolate the first table on [Unofficial progression before the IAAF](https://en.wikipedia.org/wiki/Men%27s_100_metres_world_record_progression#Unofficial_progression_before_the_IAAF). 

Next, we can use [SelectorGadget](http://selectorgadget.com/) to choose the CSS selector. In this case, I get "div+ .wikitable :nth-child(1)".


```r
m100 %>%
  html_nodes("div+ .wikitable :nth-child(1)") %>%
  html_table(fill=TRUE) 
```

```
## Error in html_table.xml_node(X[[i]], ...): html_name(x) == "table" is not TRUE
```

Uh oh! We (or, at least, I) run into an error. I won't go into details here, but we have to careful with SelectorGadget sometimes. What looks like the right selection (i.e. the highlighted stuff in yellow) might not be exactly what we're looking for. Fortunately, there's a more precise way of determing the right selectors using the "inspect web element" feature that [should be available in any modern browser](https://www.lifewire.com/get-inspect-element-tool-for-browser-756549). In this case, I'm going to use Google Chrome (either right click and then "Inspect", or **Ctrl+Shift+I**). I proceed by scrolling over the source elements until Chrome highlights the table of interest. Then right click and **Copy -> Copy selector**. 


```r
m100 %>%
  html_nodes("#mw-content-text > div > table:nth-child(8)")
```

```
## {xml_nodeset (1)}
## [1] <table class="wikitable"><tbody>\n<tr>\n<th>Time\n</th>\n<th>Athlete ...
```

Great, it worked. Now let's assign it to an object that we'll call `pre_iaaf`. We can also convert it to a data frame using the `rvest::html_table()` function. We'll also use the `fill=TRUE` option to overcome some missing column problems.


```r
pre_iaaf <-
  m100 %>%
  html_nodes("#mw-content-text > div > table:nth-child(8)") %>%
  html_table(fill=TRUE) 
class(pre_iaaf)
```

```
## [1] "list"
```

Hmmm... It turns out this is actually still a list, so let's *really* convert it to a data frame. One way to do this is by using dplyr's `bind_rows()` function, which is great for coercing (multiple) lists into a data frame.^[We'll see more examples of this once we get to the programming section of the course.] I also want to make some ggplot figures further below, so I'll just go ahead and load the whole tidyverse.


```r
## Convert list to data_frame
# pre_iaaf <- pre_iaaf[[1]] ## Would also work
library(tidyverse)

pre_iaaf <- 
  pre_iaaf %>%
  bind_rows() %>%
  as_tibble()
pre_iaaf
```

```
## # A tibble: 21 x 5
##     Time Athlete           Nationality   `Location of races`  Date         
##    <dbl> <chr>             <chr>         <chr>                <chr>        
##  1  10.8 Luther Cary       United States Paris, France        July 4, 1891 
##  2  10.8 Cecil Lee         United Kingd… Brussels, Belgium    September 25…
##  3  10.8 Etienne De Re     Belgium       Brussels, Belgium    August 4, 18…
##  4  10.8 L. Atcherley      United Kingd… Frankfurt/Main, Ger… April 13, 18…
##  5  10.8 Harry Beaton      United Kingd… Rotterdam, Netherla… August 28, 1…
##  6  10.8 Harald Anderson-… Sweden        Helsingborg, Sweden  August 9, 18…
##  7  10.8 Isaac Westergren  Sweden        Gävle, Sweden        September 11…
##  8  10.8 10.8              Sweden        Gävle, Sweden        September 10…
##  9  10.8 Frank Jarvis      United States Paris, France        July 14, 1900
## 10  10.8 Walter Tewksbury  United States Paris, France        July 14, 1900
## # … with 11 more rows
```

Let' fix the column names to get rid of spaces, etc. I'm going to use the janitor package's `clean_names()`, which is expressly built for the purpose of cleaning object names. (How else could we have done this?)


```r
library(janitor)

pre_iaaf <-
  pre_iaaf %>%
  clean_names()
pre_iaaf
```

```
## # A tibble: 21 x 5
##     time athlete           nationality   location_of_races    date         
##    <dbl> <chr>             <chr>         <chr>                <chr>        
##  1  10.8 Luther Cary       United States Paris, France        July 4, 1891 
##  2  10.8 Cecil Lee         United Kingd… Brussels, Belgium    September 25…
##  3  10.8 Etienne De Re     Belgium       Brussels, Belgium    August 4, 18…
##  4  10.8 L. Atcherley      United Kingd… Frankfurt/Main, Ger… April 13, 18…
##  5  10.8 Harry Beaton      United Kingd… Rotterdam, Netherla… August 28, 1…
##  6  10.8 Harald Anderson-… Sweden        Helsingborg, Sweden  August 9, 18…
##  7  10.8 Isaac Westergren  Sweden        Gävle, Sweden        September 11…
##  8  10.8 10.8              Sweden        Gävle, Sweden        September 10…
##  9  10.8 Frank Jarvis      United States Paris, France        July 14, 1900
## 10  10.8 Walter Tewksbury  United States Paris, France        July 14, 1900
## # … with 11 more rows
```

Hmmm. There are some potential problems for a duplicate (i.e. repeated) record for Isaac Westergren in Gävle, Sweden. One way to ID and fix these cases is to see if we can convert "athlete" into a numeric and, if so, replace these cases with the previous value.


```r
pre_iaaf <-
  pre_iaaf %>%
  mutate(athlete = ifelse(is.na(as.numeric(athlete)), athlete, lag(athlete)))
```

```
## Warning in ifelse(is.na(as.numeric(athlete)), athlete, c(NA, "Luther
## Cary", : NAs introduced by coercion
```

Lastly, let's fix the date column so that R recognises that the character string for what it actually is.


```r
library(lubridate)

pre_iaaf <-
  pre_iaaf %>%
  mutate(date = mdy(date))
pre_iaaf
```

```
## # A tibble: 21 x 5
##     time athlete             nationality   location_of_races     date      
##    <dbl> <chr>               <chr>         <chr>                 <date>    
##  1  10.8 Luther Cary         United States Paris, France         1891-07-04
##  2  10.8 Cecil Lee           United Kingd… Brussels, Belgium     1892-09-25
##  3  10.8 Etienne De Re       Belgium       Brussels, Belgium     1893-08-04
##  4  10.8 L. Atcherley        United Kingd… Frankfurt/Main, Germ… 1895-04-13
##  5  10.8 Harry Beaton        United Kingd… Rotterdam, Netherlan… 1895-08-28
##  6  10.8 Harald Anderson-Ar… Sweden        Helsingborg, Sweden   1896-08-09
##  7  10.8 Isaac Westergren    Sweden        Gävle, Sweden         1898-09-11
##  8  10.8 Isaac Westergren    Sweden        Gävle, Sweden         1899-09-10
##  9  10.8 Frank Jarvis        United States Paris, France         1900-07-14
## 10  10.8 Walter Tewksbury    United States Paris, France         1900-07-14
## # … with 11 more rows
```

Finally, we have our cleaned data frame. We can now easily plot the data if we wanted. I'm going to use (and set) the `theme_ipsum()` plotting theme from the hrbrthemes package because I like it, but this certainly isn't necessary.


```r
library(hrbrthemes) ## Just for the theme_ipsum() plot theme that I like
theme_set(theme_ipsum()) ## Set the theme for the rest of this R session

ggplot(pre_iaaf, aes(date, time)) + geom_point()
```

![](06-web-css_files/figure-html/pre_plot-1.png)<!-- -->

#### Challenge

Your turn: Download the next two tables from the same WR100m page. Combine these two new tables with the one above into a single data frame and then plot the record progression. Answer below. (No peeking until you have tried yourself first.)

.

.

.

.

.

.

.

.

.

.

.

.

.

.

.

#### Table 2: Pre-automatic timing (1912--1976)

Let's start with the second table.

```r
iaaf_76 <-
  m100 %>%
  html_nodes("#mw-content-text > div > table:nth-child(14)") %>%
  html_table(fill=TRUE) 

## Convert list to data_frame and clean the column names
iaaf_76 <- 
  iaaf_76 %>%
  bind_rows() %>%
  as_tibble() %>%
  clean_names()
```

Fill in any missing athlete data (note that we need slightly different procedure than last time --- Why?) and correct the date. 


```r
iaaf_76 <-
  iaaf_76 %>%
  mutate(athlete = ifelse(athlete=="", lag(athlete), athlete)) %>%
  mutate(date = mdy(date)) 
```

```
## Warning: 3 failed to parse.
```

It looks like some dates failed to parse because a record was broken (equaled) on the same day. E.g.


```r
iaaf_76 %>% tail(20)
```

```
## # A tibble: 20 x 8
##     time  wind  auto athlete  nationality location_of_race date       ref  
##    <dbl> <dbl> <dbl> <chr>    <chr>       <chr>            <date>     <chr>
##  1  10     2   10.2  Jim Hin… United Sta… Modesto, USA     1967-05-27 [2]  
##  2  10     1.8 NA    Enrique… Cuba        Budapest, Hunga… 1967-06-17 [2]  
##  3  10     0   NA    Paul Na… South Afri… Krugersdorp, So… 1968-04-02 [2]  
##  4  10     1.1 NA    Oliver … United Sta… Albuquerque, USA 1968-05-31 [2]  
##  5  10     2   10.2  Oliver … Charles Gr… Sacramento, USA  1968-06-20 [2]  
##  6  10     2   10.3  Oliver … Charles Gr… Roger Bambuck    NA         ""   
##  7   9.9   0.8 10.0  Jim Hin… United Sta… Sacramento, USA  1968-06-20 [2]  
##  8   9.9   0.9 10.1  Ronnie … United Sta… Sacramento, USA  1968-06-20 ""   
##  9   9.9   0.9 10.1  Charles… United Sta… Sacramento, USA  1968-06-20 ""   
## 10   9.9   0.3  9.95 Jim Hin… United Sta… Mexico City, Me… 1968-10-14 [2]  
## 11   9.9   0   NA    Eddie H… United Sta… Eugene, USA      1972-07-01 [2]  
## 12   9.9   0   NA    Eddie H… United Sta… United States    NA         ""   
## 13   9.9   1.3 NA    Steve W… United Sta… Los Angeles, USA 1974-06-21 [2]  
## 14   9.9   1.7 NA    Silvio … Cuba        Ostrava, Czecho… 1975-06-05 [2]  
## 15   9.9   0   NA    Steve W… United Sta… Siena, Italy     1975-07-16 [2]  
## 16   9.9  -0.2 NA    Steve W… ""          Berlin, Germany  1975-08-22 [2]  
## 17   9.9   0.7 NA    Steve W… ""          Gainesville, USA 1976-03-27 [2]  
## 18   9.9   0.7 NA    Steve W… Harvey Gla… Columbia, USA    1976-04-03 [2]  
## 19   9.9  NA   NA    Steve W… ""          Baton Rouge, USA 1976-05-01 [2]  
## 20   9.9   1.7 NA    Don Qua… Jamaica     Modesto, USA     1976-05-22 [2]
```

We can try to fix these cases by using the previous value. Let's test it first:


```r
iaaf_76 %>%
  mutate(date = ifelse(is.na(date), lag(date), date))
```

```
## # A tibble: 54 x 8
##     time  wind  auto athlete    nationality  location_of_race    date ref  
##    <dbl> <dbl> <dbl> <chr>      <chr>        <chr>              <dbl> <chr>
##  1  10.6  NA    NA   Donald Li… United Stat… Stockholm, Sweden -20998 [2]  
##  2  10.6  NA    NA   Jackson S… United Stat… Stockholm, Sweden -18004 [2]  
##  3  10.4  NA    NA   Charley P… United Stat… Redlands, USA     -17785 [2]  
##  4  10.4   0    NA   Eddie Tol… United Stat… Stockholm, Sweden -14756 [2]  
##  5  10.4  NA    NA   Eddie Tol… United Stat… Copenhagen, Denm… -14739 [2]  
##  6  10.3  NA    NA   Percy Wil… Canada       Toronto, Ontario… -14390 [2]  
##  7  10.3   0.4  10.4 Eddie Tol… United Stat… Los Angeles, USA  -13667 [2]  
##  8  10.3  NA    NA   Eddie Tol… Ralph Metca… Budapest, Hungary -13291 [2]  
##  9  10.3  NA    NA   Eddie Tol… Eulace Peac… Oslo, Norway      -12932 [2]  
## 10  10.3  NA    NA   Chris Ber… Netherlands  Amsterdam, Nethe… -12912 [2]  
## # … with 44 more rows
```

Whoops! Looks like all of our dates are getting converted to numbers. The reason (if you did a bit of Googling) actually has to do with the base `ifelse()` function. In this case, it's better to use the tidyverse equivalent, i.e. `if_else()`.


```r
iaaf_76 <-
  iaaf_76 %>%
  mutate(date = if_else(is.na(date), lag(date), date))
iaaf_76
```

```
## # A tibble: 54 x 8
##     time  wind  auto athlete  nationality location_of_race date       ref  
##    <dbl> <dbl> <dbl> <chr>    <chr>       <chr>            <date>     <chr>
##  1  10.6  NA    NA   Donald … United Sta… Stockholm, Swed… 1912-07-06 [2]  
##  2  10.6  NA    NA   Jackson… United Sta… Stockholm, Swed… 1920-09-16 [2]  
##  3  10.4  NA    NA   Charley… United Sta… Redlands, USA    1921-04-23 [2]  
##  4  10.4   0    NA   Eddie T… United Sta… Stockholm, Swed… 1929-08-08 [2]  
##  5  10.4  NA    NA   Eddie T… United Sta… Copenhagen, Den… 1929-08-25 [2]  
##  6  10.3  NA    NA   Percy W… Canada      Toronto, Ontari… 1930-08-09 [2]  
##  7  10.3   0.4  10.4 Eddie T… United Sta… Los Angeles, USA 1932-08-01 [2]  
##  8  10.3  NA    NA   Eddie T… Ralph Metc… Budapest, Hunga… 1933-08-12 [2]  
##  9  10.3  NA    NA   Eddie T… Eulace Pea… Oslo, Norway     1934-08-06 [2]  
## 10  10.3  NA    NA   Chris B… Netherlands Amsterdam, Neth… 1934-08-26 [2]  
## # … with 44 more rows
```


#### Table 3: Modern Era (1977 onwards)

The final table also has its share of unique complications due to row spans, etc. You can inspect the code to see what I'm doing, but I'm just going to run through it here in a single chunk.


```r
iaaf <-
  m100 %>%
  html_nodes("#mw-content-text > div > table:nth-child(19)") %>%
  html_table(fill=TRUE) 

## Convert list to data_frame and clean the column names
iaaf <- 
  iaaf %>%
  bind_rows() %>%
  as_tibble() %>%
  clean_names()

## Correct the date. 
iaaf <-
  iaaf %>%
  mutate(date = mdy(date))

## Usain Bolt's records basically all get attributed you to Asafa Powell because
## of Wikipedia row spans (same country, etc.). E.g.
iaaf %>% tail(8)
```

```
## # A tibble: 8 x 8
##    time wind   auto athlete nationality location_of_race date      
##   <dbl> <chr> <dbl> <chr>   <chr>       <chr>            <date>    
## 1  9.77 1.6    9.77 Asafa … Jamaica     Athens, Greece   2005-06-14
## 2  9.77 1.7    9.77 Justin… United Sta… Doha, Qatar      2006-05-12
## 3  9.77 1.5    9.76 Asafa … Jamaica     Gateshead, Engl… 2006-06-11
## 4  9.77 1.0    9.76 Asafa … 9.762       Zürich, Switzer… 2006-08-18
## 5  9.74 1.7    9.76 Asafa … 9.735       Rieti, Italy     2007-09-09
## 6  9.72 1.7   NA    Asafa … Usain Bolt  New York, USA    2008-05-31
## 7  9.69 0.0    9.68 Asafa … Asafa Powe… Beijing, China   2008-08-16
## 8  9.58 0.9    9.57 Asafa … Asafa Powe… Berlin, Germany  2009-08-16
## # … with 1 more variable: notes_note_2 <chr>
```

```r
## Let's fix this issue
iaaf <-
  iaaf %>%
  mutate(
    athlete = ifelse(athlete==nationality, NA, athlete),
    athlete = ifelse(!is.na(as.numeric(nationality)), NA, athlete),
    athlete = ifelse(nationality=="Usain Bolt", nationality, athlete),
    nationality = ifelse(is.na(athlete), NA, nationality),
    nationality = ifelse(athlete==nationality, NA, nationality)
    ) %>%
  fill(athlete, nationality)
```

```
## Warning in ifelse(!is.na(as.numeric(nationality)), NA, athlete): NAs
## introduced by coercion
```

#### Combined eras

Let's bind all these separate eras into a single data frame. I'll use `dplyr:: bind_rows()` again and select in the common variables only. I'll also add a new column describing which era an observation falls under.


```r
wr100 <- 
  bind_rows(
    pre_iaaf %>% select(time, athlete, nationality:date) %>% mutate(era = "Pre-IAAF"),
    iaaf_76 %>% select(time, athlete, nationality:date) %>% mutate(era = "Pre-automatic"),
    iaaf %>% select(time, athlete, nationality:date) %>% mutate(era = "Modern")
  )
wr100
```

```
## # A tibble: 99 x 7
##     time athlete nationality location_of_rac… date       era  
##    <dbl> <chr>   <chr>       <chr>            <date>     <chr>
##  1  10.8 Luther… United Sta… Paris, France    1891-07-04 Pre-…
##  2  10.8 Cecil … United Kin… Brussels, Belgi… 1892-09-25 Pre-…
##  3  10.8 Etienn… Belgium     Brussels, Belgi… 1893-08-04 Pre-…
##  4  10.8 L. Atc… United Kin… Frankfurt/Main,… 1895-04-13 Pre-…
##  5  10.8 Harry … United Kin… Rotterdam, Neth… 1895-08-28 Pre-…
##  6  10.8 Harald… Sweden      Helsingborg, Sw… 1896-08-09 Pre-…
##  7  10.8 Isaac … Sweden      Gävle, Sweden    1898-09-11 Pre-…
##  8  10.8 Isaac … Sweden      Gävle, Sweden    1899-09-10 Pre-…
##  9  10.8 Frank … United Sta… Paris, France    1900-07-14 Pre-…
## 10  10.8 Walter… United Sta… Paris, France    1900-07-14 Pre-…
## # … with 89 more rows, and 1 more variable: location_of_race <chr>
```

All that hard works deserves a nice plot, don't you think?


```r
wr100 %>%
  ggplot(aes(date, time)) + 
  geom_point(alpha = 0.7) +
  labs(
    title = "Men's 100m world record progression",
    x = "Date", y = "Time",
    caption = "Source: http://en.wikipedia.org/wiki/Men%27s_100_metres_world_record_progression"
    )
```

![](06-web-css_files/figure-html/full_plot-1.png)<!-- -->


Or, if we can just plot the modern IAFF era. 

```r
wr100 %>%
  filter(era == "Modern") %>%
  ggplot(aes(date, time)) + 
  geom_point(alpha = 0.7) +
  labs(
    title = "Men's 100m world record progression",
    subtitle = "Modern era only",
    x = "Date", y = "Time",
    caption = "Source: http://en.wikipedia.org/wiki/Men%27s_100_metres_world_record_progression"
    )
```

![](06-web-css_files/figure-html/modern_plot-1.png)<!-- -->

#### Extra: Modeling and prediction

How would you model the progression of the Men's 100 meter record over time? For example, imagine that you had to predict today's WR in 2005. How do your predictions stack up against the actual record (i.e. Usain Bolt's 9.58 time set in 2009)? How do you intepret this?

*Hint: See the `?broom::tidy()` help function for extracting refression coefients in a convenient data frame. We've already seen the `geom_smooth()` function, but for some nice tips on (visualizing) model predictions, see [Chap. 23](http://r4ds.had.co.nz/model-basics.html#visualising-models) of the R4DS book, or [Chap. 6.4](http://socviz.co/modeling.html#generate-predictions-to-graph) of the SocViz book. The generic `base::predict()` function has you covered, although the tidyverse's `modelr` package has some nice wrapper functions that you will probably find useful for this example. While it is perhaps less relevant here, the [`margins` package](https://cran.r-project.org/web/packages/margins/vignettes/Introduction.html) is very helpful if you ever want to extract marginal effects from nonlinear models (interaction terms, etc.).*