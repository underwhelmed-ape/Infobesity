---
title: "Web scraping online insect survey"
date: 2017-09-01T01:00:00Z
featured_image: '/posts/RIS-insect-survey/RIS-cover.jpeg'
author: "James"
summary: "Using R to scrape data from HTML tables"
tags: ["Web Scraping", "R"]
draft: false
---

Introduction to Rothamsted's Insect Survey
---------------------------------

Rothamsted is the UK's leading agricultural research institutes. It is
also the world's oldest agricultural institute and still maintains the
longest runnning field experiment, which has been in operation since 1856. Rothamsted is also historically significant for statisticians with
Fisher and Anscombe working there post World War 2.

The Rothamsted Insect Survey (RIS) is a programme that maintains a
network of 16 traps around the UK emptied daily and publish aggregated
weekly data tables during Aphid season. The survey reports weekly count
data of the abundance of pest Aphids and can be found
[here](https://insectsurvey.com/). 

*Note, the pages holding the survey data have since been updated and this script will not scrape as it is.*

This post is to document my process for web-scraping in R to extract data
from HTML tables over multiple pages and concatenating these into a
single time-series dataframe suitable for analysis.

### Installed Packages

    # Data Manipulation
    library(dplyr)
    library(stringr)
    library(lubridate)
    library(data.table)
    library(reshape2)

    # Web Scraping
    library(rvest)
    library(httr)

    # Data Visualisation
    library(ggplot2)
    library(Amelia) # missmap function

Accessing the Data
------------------

Each Bulletin containing the HTML table of data is accessed through
several pages of links to individual URLs. The challenge is to extract
the tables at each url over each pages.

This creates a list of the parameters to the page URLs to loop through
and scrape from.

### Pagination

This is a table of the pages that hold the URL links. Each of the
`PageURL`s will be concatenated onto the end of the base URL (`surveys`)
to access each page.

    surveys <- read_html("http://resources.rothamsted.ac.uk/insect-survey/bulletins")

    pagination <-
        data.frame(
            PageName = "Page 1",
            PageUrl = "/"
        )

    pagination <- rbind(
        pagination,
        data.frame(
            PageName = surveys %>%
                html_nodes("li.pager-item a") %>% # extracts anchor attributes from each 'next page' link
                html_attr("title") %>% # extracts title-attribute from outputs
                gsub("Go to ","", x = .) %>% # cleaning up titles
                gsub("p", "P", x = .),
            PageUrl = surveys %>%
                html_nodes("li.pager-item a") %>% # same as before
                html_attr("href") %>% # extracts the href from attributes
                gsub("insect-survey/bulletins", "", x = .))
        )

    pagination

    ##   PageName  PageUrl
    ## 1   Page 1        /
    ## 2   Page 2 /?page=1
    ## 3   Page 3 /?page=2
    ## 4   Page 4 /?page=3
    ## 5   Page 5 /?page=4
    ## 6   Page 6 /?page=5

### Extracting elements from Base URL

With the pagination table, I can then loop through each page and extract
attributes from each link, using another loop, including the heading of
each Bulletin and the URL to the tables. This is stored in a dataframe
of all the links.

Not all the links contain weekly bulletins. The links that are
bulletins, containing the data, have a common naming pattern within
their title `"Bulletin No: "`. I used `grep` to only bring back those
instances that contain that pattern.

    all_bulletins <- data.frame() # create empty dataframe for the bulletin list.

    # Loop through each page in pagination dataframe, extract title and url of each link.

    for (page in 1:nrow(pagination)) {
        page_url = paste("http://resources.rothamsted.ac.uk/insect-survey/bulletins", pagination$PageUrl[page], sep = "")
        parsed_page = read_html(page_url)
        bulletins = data.frame(BulletinName = parsed_page %>%
                                   html_nodes("h4.title") %>%
                                   html_text(),
                               BulletinUrl = parsed_page %>%
                                   html_nodes("h4.title a") %>%
                                   html_attr("href")) %>%
            dplyr::filter(
                 grepl(pattern = "Bulletin No: ", BulletinName) # filters to only those with datasets
            )
        all_bulletins = rbind(all_bulletins, bulletins) # create dataframe concatenating all bulletins
        all_bulletins <- droplevels(all_bulletins) # removes unused levels subsetted out
    }

    rm(page, page_url, parsed_page, surveys)

    all_bulletins %>% glimpse

    ## Observations: 122
    ## Variables: 2
    ## $ BulletinName <fctr> Bulletin No: 22. 21 August - 27 August 2017, Bul...
    ## $ BulletinUrl  <fctr> /insect-survey-bulletins/bulletin-no-22-21-augus...

### Inspecting the Dataframe

The web scraping has extracted the name and the URL that links to the
data for each Bulletin over the 6 pages of results. As I will need a way
of knowing which data comes from each Bulletin after putting them all
together, I will extract information from the names here using Regular
Expressions.

    # Regex to extract constituents from bulletin names

    all_bulletins %>%
        mutate(BulletinNo = str_extract(string = all_bulletins$BulletinName, "Bulletin No: \\d+"),
               year = str_extract(string = all_bulletins$BulletinName, "\\d+$"),
               date_range = str_replace(string = all_bulletins$BulletinName,
                pattern = "Bulletin No: \\d+\\. ",
                replacement = "")
               ) -> all_bulletins

    tail(all_bulletins)

    ##                            BulletinName
    ## 117     Bulletin No: 7. 12 May - 18 May
    ## 118     Bulletin No: 6. 05 May - 11 May
    ## 119   Bulletin No: 5. 28 April - 04 May
    ## 120       Bulletin No: 2. 07 - 13 April
    ## 121       Bulletin No: 3. 14 - 20 April
    ## 122 Bulletin No: 1. 31 March - 06 April
    ##                                                  BulletinUrl
    ## 117     /insect-survey-bulletins/bulletin-no-7-12-may-18-may
    ## 118     /insect-survey-bulletins/bulletin-no-6-05-may-11-may
    ## 119   /insect-survey-bulletins/bulletin-no-5-28-april-04-may
    ## 120       /insect-survey-bulletins/bulletin-no-2-07-13-april
    ## 121       /insect-survey-bulletins/bulletin-no-3-14-20-april
    ## 122 /insect-survey-bulletins/bulletin-no-1-31-march-06-april
    ##         BulletinNo year          date_range
    ## 117 Bulletin No: 7 <NA>     12 May - 18 May
    ## 118 Bulletin No: 6 <NA>     05 May - 11 May
    ## 119 Bulletin No: 5 <NA>   28 April - 04 May
    ## 120 Bulletin No: 2 <NA>       07 - 13 April
    ## 121 Bulletin No: 3 <NA>       14 - 20 April
    ## 122 Bulletin No: 1 <NA> 31 March - 06 April

If we view this now, I have extracted and added as separate factors, the
Bulletin number, the year and the range of dates that the data covers.
This isn't the cleanest data, the earliest reports made in 2014 do not
have the year within the title, also the format of some of the date
ranges vary in how they were written.

Adding 2014 to the year manually:

    all_bulletins$year[is.na(all_bulletins$year)] <- "2014"

### Creating an Aphid Collection Date

This data is a time series and so I need to be able to extract a clean
date from each title. As mentioned before, the formats are not always
consistent:

    str_extract(string = all_bulletins$date_range,
                pattern = "\\d+ \\w+ ")

    ##   [1] "21 August "    "14 August "    "07 August "    "31 July "     
    ##   [5] "24 July "      "17 July "      "10 July "      "02 July "     
    ##   [9] "26 June "      "19 June "      "12 June "      "05 June "     
    ##  [13] "29 May "       "22 May "       "15 May "       "08 May "      
    ##  [17] "01 May "       "24 April "     "17 April "     "10 April "    
    ##  [21] "03 April "     "27 March "     "07 November "  "06 November "
    ##  [25] "24 October "   "17 October "   "10 October "   "03 October "  
    ##  [29] "26 September " "19 September " "12 September " "05 September "
    ##  [33] "29 August "    "22 August "    "15 August "    "08 August "   
    ##  [37] "01 August "    "25 July "      "18 July "      "11 July "     
    ##  [41] "04 July "      "27 June "      "20 June "      "13 June "     
    ##  [45] "06 June "      "30 May "       "23 May "       "16 May "      
    ##  [49] "09 May "       "02 May "       "25 April "     "18 April "    
    ##  [53] "11 April "     "04 April "     "28 March "     "21 March "    
    ##  [57] "16 November "  "09 November "  "02 November "  "26 October "  
    ##  [61] "19 October "   "12 October "   "05 October "   "28 September "
    ##  [65] "21 September " "14 September " "07 September " "31 August "   
    ##  [69] "24 August "    "17 August "    "10 August "    "03 August "   
    ##  [73] "27 July "      "20 July "      "13 July "      "06 July "     
    ##  [77] "29 June "      "22 June "      "15 June "      "08 June "     
    ##  [81] "01 June "      "25 May "       "18 May "       "11 May "      
    ##  [85] "04 May "       "27 April "     "20 April "     "13 April "    
    ##  [89] "06 April "     "17 November "  "10 November "  "03 November "
    ##  [93] "27 October "   "20 October "   "13 October "   "06 October "  
    ##  [97] "29 September " "22 September " "15 September " "08 September "
    ## [101] "01 September " "25 August "    "18 August "    "11 August "   
    ## [105] "04 August "    "28 July "      "21 July "      "14 July "     
    ## [109] "07 July "      "30 June "      "23 June "      "16 June "     
    ## [113] "09 June "      "02 June "      "26 May "       "19 May "      
    ## [117] "12 May "       "05 May "       "28 April "     NA             
    ## [121] NA              "31 March "

This shows that the naming format is not constant resulting in two
outputs with NA. As such I extract each element individually and
concatenate them to form a date that is the beginning of each data
collection exercise.

    paste(
    # Match first numbers in each string
    str_extract(string = all_bulletins$date_range,
                pattern = "\\d+"),
    # Matches the month at beginning of collection
    str_extract(string = all_bulletins$date_range,
                pattern = "[:alpha:]+"),
    # already extracted the year
    all_bulletins$year,
    sep = " "
    )

    ##   [1] "21 August 2017"    "14 August 2017"    "07 August 2017"   
    ##   [4] "31 July 2017"      "24 July 2017"      "17 July 2017"     
    ##   [7] "10 July 2017"      "02 July 2017"      "26 June 2017"     
    ##  [10] "19 June 2017"      "12 June 2017"      "05 June 2017"     
    ##  [13] "29 May 2017"       "22 May 2017"       "15 May 2017"      
    ##  [16] "08 May 2017"       "01 May 2017"       "24 April 2017"    
    ##  [19] "17 April 2017"     "10 April 2017"     "03 April 2017"    
    ##  [22] "27 March 2017"     "07 November 2016"  "31 October 2016"  
    ##  [25] "24 October 2016"   "17 October 2016"   "10 October 2016"  
    ##  [28] "03 October 2016"   "26 September 2016" "19 September 2016"
    ##  [31] "12 September 2016" "05 September 2016" "29 August 2016"   
    ##  [34] "22 August 2016"    "15 August 2016"    "08 August 2016"   
    ##  [37] "01 August 2016"    "25 July 2016"      "18 July 2016"     
    ##  [40] "11 July 2016"      "04 July 2016"      "27 June 2016"     
    ##  [43] "20 June 2016"      "13 June 2016"      "06 June 2016"     
    ##  [46] "30 May 2016"       "23 May 2016"       "16 May 2016"      
    ##  [49] "09 May 2016"       "02 May 2016"       "25 April 2016"    
    ##  [52] "18 April 2016"     "11 April 2016"     "04 April 2016"    
    ##  [55] "28 March 2016"     "21 March 2016"     "16 November 2015"
    ##  [58] "09 November 2015"  "02 November 2015"  "26 October 2015"  
    ##  [61] "19 October 2015"   "12 October 2015"   "05 October 2015"  
    ##  [64] "28 September 2015" "21 September 2015" "14 September 2015"
    ##  [67] "07 September 2015" "31 August 2015"    "24 August 2015"   
    ##  [70] "17 August 2015"    "10 August 2015"    "03 August 2015"   
    ##  [73] "27 July 2015"      "20 July 2015"      "13 July 2015"     
    ##  [76] "06 July 2015"      "29 June 2015"      "22 June 2015"     
    ##  [79] "15 June 2015"      "08 June 2015"      "01 June 2015"     
    ##  [82] "25 May 2015"       "18 May 2015"       "11 May 2015"      
    ##  [85] "04 May 2015"       "27 April 2015"     "20 April 2015"    
    ##  [88] "13 April 2015"     "06 April 2015"     "17 November 2014"
    ##  [91] "10 November 2014"  "03 November 2014"  "27 October 2014"  
    ##  [94] "20 October 2014"   "13 October 2014"   "06 October 2014"  
    ##  [97] "29 September 2014" "22 September 2014" "15 September 2014"
    ## [100] "08 September 2014" "01 September 2014" "25 August 2014"   
    ## [103] "18 August 2014"    "11 August 2014"    "04 August 2014"   
    ## [106] "28 July 2014"      "21 July 2014"      "14 July 2014"     
    ## [109] "07 July 2014"      "30 June 2014"      "23 June 2014"     
    ## [112] "16 June 2014"      "09 June 2014"      "02 June 2014"     
    ## [115] "26 May 2014"       "19 May 2014"       "12 May 2014"      
    ## [118] "05 May 2014"       "28 April 2014"     "07 April 2014"    
    ## [121] "14 April 2014"     "31 March 2014"

Adding this into the `all_bulletins` dataframe as a new factor, and give
this a date fomat.

    all_bulletins %>%
        mutate(week_start =
                   paste(
    str_extract(string = all_bulletins$date_range,
                pattern = "\\d+"),
    # Matches the Week beginning month
    str_extract(string = all_bulletins$date_range,
                pattern = "[:alpha:]+"),
    # already extracted the year
    all_bulletins$year,
    sep = " ")
    ) -> all_bulletins

    all_bulletins$week_start <- dmy(all_bulletins$week_start)

    str(all_bulletins)

    ## 'data.frame':    122 obs. of  6 variables:
    ##  $ BulletinName: Factor w/ 122 levels "Bulletin No: 1. 27 March - 02 April 2017",..: 15 14 13 11 10 9 8 7 6 5 ...
    ##  $ BulletinUrl : Factor w/ 122 levels "/insect-survey-bulletins/bulletin-no-1-27-march-02-april-2017",..: 15 14 13 11 10 9 8 7 6 5 ...
    ##  $ BulletinNo  : chr  "Bulletin No: 22" "Bulletin No: 21" "Bulletin No: 20" "Bulletin No: 19" ...
    ##  $ year        : chr  "2017" "2017" "2017" "2017" ...
    ##  $ date_range  : chr  "21 August - 27 August 2017" "14 August - 20 August 2017" "07 August - 13 August 2017" "31 July - 06 August 2017" ...
    ##  $ week_start  : Date, format: "2017-08-21" "2017-08-14" ...

Extracting the Data
-------------------

Everything up until now has been preparing for extracting the data. To
concatenate all the data together for analysis, a list is created and
the data from each table is scraped and placed into a dataframe and
placed within the list. Each dataframe in this contains the table
extracted from the site location.

This list is then concatenated together using `dplyr::bind_rows()`. The
base function `rbind()` requires columns within each table to be the
same. In this case, over such a timescale the locations of the insect
traps may not stay constant over the years. The function `bind_rows()`
will retain all columns and place 'NA' in the spaces.

    data.list = list()

    for(index in (1:nrow(all_bulletins))) {
        tryCatch({
            paste("http://resources.rothamsted.ac.uk",all_bulletins$BulletinUrl[index], sep = "") %>%
            read_html %>%
            html_table(., header = TRUE) %>%
            do.call(cbind, .) -> dat

        dat[is.na(dat)] <- 0 # assigns all NAs to 0

        dat %>%
            mutate(year = all_bulletins$year[index],
                   bulletin.number = all_bulletins$BulletinNo[index],
                   week_start = all_bulletins$week_start[index]) -> dat

        dat$index <- index

        data.list[[index]] <- dat
        }, error=function(e){})
    }

    ## Warning: closing unused connection 5 (http://resources.rothamsted.ac.uk/
    ## insect-survey-bulletins/bulletin-no-34-07-november-13-november-2016)

This outputs a warning about closing an unused connection with the
number and the URL. This informs us that bulletin number 34 from the 7th
November to 13th November 2016 could not be accessed. This was the
purpose of the `tryCatch`, without which the loop would stop executing
the script after receiving the error.

    all.data <- do.call(bind_rows, data.list)

Within this, I am assuming that where cells within the table are blank,
no aphids of that specied were found and have been replaced with a `0`.
This is to differentiate when a location is not available / functioning
which will contain `NA`.

Renaming some columns

    data.table::setnames(all.data,
                         old = c('Var.1','year', 'bulletin.number', 'index'),
                         new = c("Aphid_sp", "Year", "Bulletin_Number", "Index")
                         )

In the data there are observations that lists the number of part catches
and number of days of catching. Removing these.

    all.data %>%
        filter(Aphid_sp != "Days") %>%
        filter(Aphid_sp != "Part Catches") -> all.data

### Inspecting the Final Dataset

    summary(all.data)

    ##    Aphid_sp               El                 D                 G          
    ##  Length:2541        Min.   :   0.000   Min.   :  0.000   Min.   :   0.00  
    ##  Class :character   1st Qu.:   0.000   1st Qu.:  0.000   1st Qu.:   0.00  
    ##  Mode  :character   Median :   0.000   Median :  0.000   Median :   0.00  
    ##                     Mean   :   6.504   Mean   :  7.537   Mean   :  20.74  
    ##                     3rd Qu.:   0.000   3rd Qu.:  1.000   3rd Qu.:   2.00  
    ##                     Max.   :1048.000   Max.   :996.000   Max.   :6156.00  
    ##                                                                           
    ##        Ay                N                 Y                P           
    ##  Min.   :  0.000   Min.   :   0.00   Min.   :  0.00   Min.   :    0.00  
    ##  1st Qu.:  0.000   1st Qu.:   0.00   1st Qu.:  0.00   1st Qu.:    0.00  
    ##  Median :  0.000   Median :   0.00   Median :  0.00   Median :    0.00  
    ##  Mean   :  3.731   Mean   :  10.88   Mean   : 11.75   Mean   :   74.06  
    ##  3rd Qu.:  0.000   3rd Qu.:   1.00   3rd Qu.:  4.00   3rd Qu.:    3.00  
    ##  Max.   :667.000   Max.   :1895.00   Max.   :857.00   Max.   :18287.00  
    ##                                      NA's   :693                        
    ##        K                 BB                We                H          
    ##  Min.   :   0.00   Min.   :   0.00   Min.   :   0.00   Min.   :   0.00  
    ##  1st Qu.:   0.00   1st Qu.:   0.00   1st Qu.:   0.00   1st Qu.:   0.00  
    ##  Median :   0.00   Median :   0.00   Median :   0.00   Median :   0.00  
    ##  Mean   :  13.18   Mean   :  14.23   Mean   :  12.29   Mean   :  12.58  
    ##  3rd Qu.:   4.00   3rd Qu.:   3.00   3rd Qu.:   3.00   3rd Qu.:   3.00  
    ##  Max.   :1847.00   Max.   :1942.00   Max.   :1368.00   Max.   :3053.00  
    ##                                                                         
    ##        RT                 Wr                SP                W          
    ##  Min.   :   0.000   Min.   :   0.00   Min.   :  0.000   Min.   :  0.000  
    ##  1st Qu.:   0.000   1st Qu.:   0.00   1st Qu.:  0.000   1st Qu.:  0.000  
    ##  Median :   0.000   Median :   0.00   Median :  0.000   Median :  0.000  
    ##  Mean   :   9.289   Mean   :  12.45   Mean   :  3.553   Mean   :  6.149  
    ##  3rd Qu.:   3.000   3rd Qu.:   3.00   3rd Qu.:  1.000   3rd Qu.:  2.000  
    ##  Max.   :1627.000   Max.   :1381.00   Max.   :639.000   Max.   :510.000  
    ##                                                                          
    ##        SX             Year           Bulletin_Number   
    ##  Min.   :  0.00   Length:2541        Length:2541       
    ##  1st Qu.:  0.00   Class :character   Class :character  
    ##  Median :  0.00   Mode  :character   Mode  :character  
    ##  Mean   :  7.58                                        
    ##  3rd Qu.:  3.00                                        
    ##  Max.   :664.00                                        
    ##                                                        
    ##    week_start             Index              AB      
    ##  Min.   :2014-03-31   Min.   :  1.00   Min.   :0     
    ##  1st Qu.:2014-11-03   1st Qu.: 32.00   1st Qu.:0     
    ##  Median :2015-10-12   Median : 62.00   Median :0     
    ##  Mean   :2015-11-24   Mean   : 61.82   Mean   :0     
    ##  3rd Qu.:2016-09-05   3rd Qu.: 92.00   3rd Qu.:0     
    ##  Max.   :2017-08-21   Max.   :122.00   Max.   :0     
    ##                                        NA's   :2247

    Amelia::missmap(all.data,
                    col = c("steelblue", "lightgrey"),
                    y.labels = c(2500, 2250, 2000, 1750, 1500, 1250, 1000, 750, 500, 250, 1),
                    y.at = c(1, 250, 500, 750, 1000, 1250, 1500, 1750, 2000, 2250, 2500)+20,
                    main = "Missing data Map")


![image](/posts/RIS-insect-survey/images/missing_map.png)


This indicates that the collection traps at `AB` were not available for
chunks of time at the beginning of the period. Also the collection at
`Y` location was discontinued.

Plotting Aphid species counts for Rothamsted Tower.

    all.data %>%
        ggplot(aes(x = week_start, y = RT)) +
        geom_point() +
        theme_bw() +
        labs(
            title = "Aphid Species Captured at Rothamsted Tower",
            x = "Collection Date",
            y = "Aphid Counts")


![image](/posts/RIS-insect-survey/images/results.png)



From this point the data can be manipulated and cleaned as required for
analysis.

Session Info
------------

    ## R version 3.3.3 (2017-03-06)
    ## Platform: x86_64-apple-darwin13.4.0 (64-bit)
    ## Running under: macOS Sierra 10.12.6
    ##
    ## locale:
    ## [1] en_GB.UTF-8/en_GB.UTF-8/en_GB.UTF-8/C/en_GB.UTF-8/en_GB.UTF-8
    ##
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ##
    ## other attached packages:
    ##  [1] Amelia_1.7.4      Rcpp_0.12.12      ggplot2_2.2.1    
    ##  [4] httr_1.2.1        rvest_0.3.2       xml2_1.1.1       
    ##  [7] reshape2_1.4.2    data.table_1.10.0 lubridate_1.6.0  
    ## [10] stringr_1.2.0     dplyr_0.5.0      
    ##
    ## loaded via a namespace (and not attached):
    ##  [1] knitr_1.16       magrittr_1.5     munsell_0.4.3    colorspace_1.3-2
    ##  [5] R6_2.2.0         plyr_1.8.4       tools_3.3.3      grid_3.3.3      
    ##  [9] gtable_0.2.0     DBI_0.5-1        selectr_0.3-1    htmltools_0.3.6
    ## [13] lazyeval_0.2.0   yaml_2.1.14      assertthat_0.1   rprojroot_1.2   
    ## [17] digest_0.6.12    tibble_1.2       curl_2.3         evaluate_0.10.1
    ## [21] rmarkdown_1.6    labeling_0.3     stringi_1.1.5    scales_0.4.1    
    ## [25] backports_1.0.5  XML_3.98-1.5     foreign_0.8-67