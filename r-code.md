# Big-ish Data Workflow in R
Ryan Kelly  
April 11, 2015  


This is an R version of a recent article [Big data analytics with Pandas and SQLite](https://plot.ly/ipython-notebooks/big-data-analytics-with-pandas-and-sqlite/) by [plotly](https://plot.ly/feed/). I'd argue this is 'medium' data, however, I understand the need to generate a nice headline. For the R folks, I also expand on alternative packages and methods that may improve the workflow. 

This document is not meant to make any claims that R is better or faster than Python for data analysis, I myself use both languages daily. The plotly article simply provides an opportunity to compare the two languages. 

The [data](https://data.cityofnewyork.us/Social-Services/311-Service-Requests-from-2010-to-Present/erm2-nwe9) from this article is updated daily, therefore my version of the file is quite a bit larger at 4.63 GB. Therefore, expect the charts to return different results compared to the article. 

Here are the details quoted from the article...
<hr>

This notebook explores a 3.9GB CSV file containing NYC's 311 complaints since 2003. It's the most popular data set in NYC's open data portal.

This notebook is a primer on out-of-memory data analysis with 

- [pandas](https://www.google.ca/search?client=safari&rls=en&q=pandas+python&ie=UTF-8&oe=UTF-8&gfe_rd=cr&ei=O6ApVZClJIaN8Qf66IA4) : A library with easy-to-use data structures and data analysis tools. Also, interfaces to out-of-memory databases like SQLite.
- [IPython notebook](http://ipython.org/notebook.html) : An interface for writing and sharing python code, text, and plots.
- [SQLite](https://www.sqlite.org) : An self-contained, server-less database that's easy to set-up and query from Pandas.
- [plotly](https://plot.ly) : A platform for publishing beautiful, interactive graphs from Python to the web.

The dataset is too large to load into a Pandas dataframe. So, instead we'll perform out-of-memory aggregations with SQLite and load the result directly into a dataframe with Panda's iotools . It's pretty easy to stream a CSV into SQLite and SQLite requires no setup. The SQL query language is pretty intuitive coming from a Pandas mindset.

<hr>

Since we are using R, we will be substituting the python packages with the closest R counterpart, this is of course subjective.

- Instead of [pandas](https://www.google.ca/search?client=safari&rls=en&q=pandas+python&ie=UTF-8&oe=UTF-8&gfe_rd=cr&ei=O6ApVZClJIaN8Qf66IA4), we will be using [dplyr](https://www.google.ca/search?client=safari&rls=en&q=dplyr+github&ie=UTF-8&oe=UTF-8&gfe_rd=cr&ei=T6ApVYqcJ4aN8Qf66IA4)

- Instead of [IPython notebook](http://ipython.org/notebook.html) I have published this document using [knitr](https://github.com/yihui/knitr) / [Rmarkdown](http://rmarkdown.rstudio.com) via [RStudio](http://www.rstudio.com)

- I will maintain the use of SQLite, and [plotly for R](https://plot.ly/r/getting-started/) & [ggplot2](http://ggplot2.org). However, for time series charts, I highly recommend the R package [dygraphs](http://rstudio.github.io/dygraphs/), which provides similar interactive plotting, without requiring an internet connection. 

> Where possible I will outline to opporunities to substitute different R packages that in my opinion may improve the workflow. 


## The Data Workflow

### Setting up plotly in R

[plotly for R](https://plot.ly/r/getting-started/) is not yet available on [CRAN](http://cran.r-project.org). We can instead download it from the [rOpenSci](https://github.com/ropensci) github repo.



```r
install.packages("devtools")
library("devtools")
install_github("ropensci/plotly")
```


```r
library(plotly)
```

You'll need to create an account to connect to the plotly API. Or you could just stick with the default ggplot2 graphics. 


```r
set_credentials_file("DemoAccount", "lr1c37zw81") ## Replace contents with your API Key
```



### Importing the CSV in SQLite

This defeats the purpose of this article, but **R can** load this data into memory *easily*.  If your computer resources permit it, the data will be much faster to operate on in-memory compared to an SQL database on disk. 8GB of RAM would be plenty in this case. In the spirit of a true comparison, I will replicate the on-disk analysis approach using an SQLite database, however, I will show a few benchmark's using the data in memory using `data.table` along the way. 

## On-Disk Analysis in R with dplyr

While `dplyr` is capable of writing to databases, the data still must flow through R, which would probably be considered cheating in this case. The alternative is to simply to create a database and import the csv using the command line. Please create a pull request for [this github repository](https://github.com/RMDK/bigish-data-in-R) if you have any other suggestions. 

Here is the code I typed into the command line. Pretty easy right? This assumes you have sqlite3 installed and available on your PATH variable (hence accessible via terminal).


```text
$ sqlite3 data.db # Create your database
$.databases       # Show databases to make sure it works
$.mode csv        
$.import <filename> <tablename>
# Where filename is the name of the csv & tablename is the name of the new database table
$.quit 
```

Let's also load the data into memory so we can compare in memory operations along the way. Here is a crude benchmark of file I/O in R. [readr](https://github.com/hadley/readr) a recently released alternative to `read.csv` should also eat through this data quickly.  



```r
library(readr)
# data.table, selecting a subset of columns
time_data.table <- system.time(fread('/users/ryankelly/NYC_data.csv', 
                   select = c('Agency', 'Created Date','Closed Date', 'Complaint Type', 'Descriptor', 'City'), 
                   showProgress = T))
# Default data.table
time_data.table_full <- system.time(fread('/users/ryankelly/NYC_data.csv',
                                            showProgress = T))
# Default readr
time_readr <- system.time(read_csv('/users/ryankelly/NYC_data.csv'))

# Default base R (really slow - don't recommend running)
# time_base_r <- system.time(read.csv('/users/ryankelly/NYC_data.csv'))
```




```r
data.frame(rbind(time_data.table, time_data.table_full, time_readr))
```

```
##                      user.self sys.self elapsed user.child sys.child
## time_data.table         67.654    5.383  84.308          0         0
## time_data.table_full   213.371    5.738 226.433          0         0
## time_readr             290.729    7.542 304.974          0         0
```

I will be using data.table to read in the data. The `fread` function has the really nice ability to select the columns we want to read in, this speeds up the read quite a bit. The plotly article disregards all but the following columns from the dataset. 


```r
library(data.table)

dt <- fread('/users/ryankelly/NYC_data.csv', 
            select = c('Agency', 'Created Date','Closed Date', 'Complaint Type', 'Descriptor', 'City'), 
            showProgress = F)

# Rename columns to remove spaces
setnames(dt, 'Created Date', 'CreatedDate')
setnames(dt, 'Closed Date', 'ClosedDate')
setnames(dt, 'Complaint Type', 'ComplaintType')
```


### About dplyr

> Switching between languages is cognitively challenging (especially because R and SQL are so perilously similar), so dplyr allows you to write R code that is automatically translated to SQL. The goal of dplyr is not to replace every SQL function with an R function: that would be difficult and error prone. Instead, dplyr only generates SELECT statements, the SQL you write most often as an analyst.
- [dplyr databases](http://cran.rstudio.com/web/packages/dplyr/vignettes/databases.html)

dplyr is pretty fantastic because you can use the *same* syntax to:

- Write dataframes to databases
- Read from databases into memory
- Operate on database via SQL wrappers
- Operate on memory

In the event you need to run a more sophisticated query or if you prefer plain SQL, you can also simply pass in an SQL statement as a string, or generate your own sql wrappers. dplyr supports: sqlite, mysql, postgresql, and google’s bigquery. You will see that dplyr's syntax is very similar to SQL.

For a more information about dplyr, see these two tutorials, which I borrow from heavily:

- [dplyr for databases](http://cran.rstudio.com/web/packages/dplyr/vignettes/databases.html)
- [dplyr tutorial](http://cran.rstudio.com/web/packages/dplyr/vignettes/introduction.html)

By default, dplyr queries will only pull the first 10 rows from the database, and never pulls data into R unless you ask for it directly. 



```r
library(dplyr)      ## Will be used for pandas replacement

# Connect to the database
db <- src_sqlite('/users/ryankelly/data.db')
db
```

```
## src:  sqlite 3.8.6 [/users/ryankelly/data.db]
## tbls: NYC_data
```

```r
# Connect to the table of interest, which I called NYC_data
data <- tbl(db, 'NYC_data')
# We can pass SQL directly here to select only the columns of interest
# I 'rename' the columns with spaces to be able to properly access them in dplyr
data <- tbl(db, sql('SELECT Agency, "Created Date" AS CreatedDate, 
                    "Closed Date" AS ClosedDate, "Complaint Type" AS ComplaintType,  Descriptor, City
                    FROM NYC_data'))
```

<br>

 Once we have to do a more expensive query, the data become pretty slow to work with. **This would be apparent in both Python or R**. This is why you would rather operate on the data in-memory. The two best choices for data manipulation (besides base R) are:

- data.table
- dplyr


Remember, the following analysis is completed using dplyr as a wrapper for SQL statements, operating on-disk. Use `explain()` to expose the SQL commands in your statements. You can simply use raw SQL strings if you prefer. *I will also show how to run the same commands in `data.table` in the comments.*
 
 
 I am not going to benchmark every query, as there are many [examples](https://github.com/Rdatatable/data.table/wiki/Benchmarks-:-Grouping) of the speed advantages of data.table. I prefer data.table for both the speed boost, and compact syntax. For a first look at data.table, take a look at this [cheatsheet](https://s3.amazonaws.com/assets.datacamp.com/img/blog/data+table+cheat+sheet.pdf).
 
 
<br>

#### Preview the data




```r
head(data)
```

```
##   Agency            CreatedDate ClosedDate           ComplaintType
## 1   NYPD 04/11/2015 02:13:04 AM            Noise - Street/Sidewalk
## 2   DFTA 04/11/2015 02:12:05 AM            Senior Center Complaint
## 3   NYPD 04/11/2015 02:11:46 AM                 Noise - Commercial
## 4   NYPD 04/11/2015 02:11:02 AM            Noise - Street/Sidewalk
## 5   NYPD 04/11/2015 02:10:45 AM            Noise - Street/Sidewalk
## 6   NYPD 04/11/2015 02:09:07 AM            Noise - Street/Sidewalk
##         Descriptor     City
## 1 Loud Music/Party BROOKLYN
## 2              N/A ELMHURST
## 3 Loud Music/Party  JAMAICA
## 4     Loud Talking BROOKLYN
## 5 Loud Music/Party NEW YORK
## 6     Loud Talking BROOKLYN
```

<br>

#### Select just a couple of columns


```r
# dt[, .(ComplaintType, Descriptor, Agency)]

q <- data %>% select(ComplaintType, Descriptor, Agency)
head(q)
```

```
##             ComplaintType       Descriptor Agency
## 1 Noise - Street/Sidewalk Loud Music/Party   NYPD
## 2 Senior Center Complaint              N/A   DFTA
## 3      Noise - Commercial Loud Music/Party   NYPD
## 4 Noise - Street/Sidewalk     Loud Talking   NYPD
## 5 Noise - Street/Sidewalk Loud Music/Party   NYPD
## 6 Noise - Street/Sidewalk     Loud Talking   NYPD
```

<br>

#### Limit the number of items retrieved


```r
# dt[, .(ComplaintType, Descriptor, Agency)][1:10]

q <- data %>% select(ComplaintType, Descriptor, Agency)
head(q, n = 10)
```

```
##              ComplaintType                    Descriptor Agency
## 1  Noise - Street/Sidewalk              Loud Music/Party   NYPD
## 2  Senior Center Complaint                           N/A   DFTA
## 3       Noise - Commercial              Loud Music/Party   NYPD
## 4  Noise - Street/Sidewalk                  Loud Talking   NYPD
## 5  Noise - Street/Sidewalk              Loud Music/Party   NYPD
## 6  Noise - Street/Sidewalk                  Loud Talking   NYPD
## 7       Noise - Commercial              Loud Music/Party   NYPD
## 8   HPD Literature Request The ABCs of Housing - Spanish    HPD
## 9  Noise - Street/Sidewalk                  Loud Talking   NYPD
## 10        Street Condition       Plate Condition - Noisy    DOT
```

<br>

#### Filter rows with WHERE


```r
# dt[Agency == 'NYPD', .(ComplaintType, Descriptor, Agency)]

q <- data %>% 
          select(ComplaintType, Descriptor, Agency) %>% 
          filter(Agency == "NYPD")
head(q)
```

```
##             ComplaintType       Descriptor Agency
## 1 Noise - Street/Sidewalk Loud Music/Party   NYPD
## 2      Noise - Commercial Loud Music/Party   NYPD
## 3 Noise - Street/Sidewalk     Loud Talking   NYPD
## 4 Noise - Street/Sidewalk Loud Music/Party   NYPD
## 5 Noise - Street/Sidewalk     Loud Talking   NYPD
## 6      Noise - Commercial Loud Music/Party   NYPD
```

<br>

#### Filter multiple values in a column with WHERE and IN 


```r
# dt[Agency == 'NYPD' | Agency == 'DOB', .(ComplaintType, Descriptor, Agency)]

q <- data %>% select(ComplaintType, Descriptor, Agency) %>% 
               filter(Agency %in% c('DOB', 'NYPD'))
head(q)
```

```
##             ComplaintType       Descriptor Agency
## 1 Noise - Street/Sidewalk Loud Music/Party   NYPD
## 2      Noise - Commercial Loud Music/Party   NYPD
## 3 Noise - Street/Sidewalk     Loud Talking   NYPD
## 4 Noise - Street/Sidewalk Loud Music/Party   NYPD
## 5 Noise - Street/Sidewalk     Loud Talking   NYPD
## 6      Noise - Commercial Loud Music/Party   NYPD
```

<br>

#### Find the unique values in a column with DISTINCT


```r
# dt[, unique(City)]

q <- data %>% select(City) %>% distinct()
head(q)
```

```
##       City
## 1 BROOKLYN
## 2 ELMHURST
## 3  JAMAICA
## 4 NEW YORK
## 5         
## 6  BAYSIDE
```

<br>

#### Query value counts with COUNT(*) and GROUP BY


```r
# dt[, .(No.Complaints = .N), Agency]
#setkey(dt, No.Complaints) # setkey index's the data

q <- data %>% select(Agency) %>% group_by(Agency) %>% summarise(No.Complaints = n())
head(q)
```

```
##   Agency No.Complaints
## 1  3-1-1         22499
## 2    ACS             3
## 3    AJC             7
## 4    ART             3
## 5    CAU             8
## 6   CCRB             9
```

<br>

#### Order the results with ORDER and -


```r
# dt[, .(No.Complaints = .N), Agency]
#setkey(dt, No.Complaints) # setkey index's the data

q <- data %>% select(Agency) %>% 
              group_by(Agency) %>% 
              summarise(No.Complaints = n()) %>%
              arrange(-No.Complaints)

# Pull the data out of memory to plot it
q <- collect(q)

# Convert to ordered factor to maintain order in plot
q$Agency <- factor(q$Agency, levels = q$Agency, ordered = T)

library(ggplot2)
# Plot top 50
plt <- ggplot(q[1:50,], aes(Agency, No.Complaints)) + 
        geom_bar(stat= 'identity') +
        theme_minimal() + theme(axis.text.x = element_text(angle = 45, hjust = 1))

# Convert to plotly object
py <- plotly()
py$ggplotly(plt, session='knitr')
```

<iframe height="600" id="igraph" scrolling="no" seamless="seamless"
				src="https://plot.ly/~rmdk/84" width="600" frameBorder="0"></iframe>

<br> 

#### Heat / Hot Water is the most common complaint


```r
# dt[, .(No.Complaints = .N), ComplaintType]
#setkey(dt, No.Complaints) # setkey index's the data

q <- data %>% select(ComplaintType, Agency) %>% 
              group_by(ComplaintType) %>% 
              summarise(No.Complaints = n()) %>%
              arrange(-No.Complaints)

# Pull the data out of memory to plot it
q <- collect(q)

# Convert to ordered factor to maintain order in plot
q$ComplaintType <- factor(q$ComplaintType, levels = q$ComplaintType, ordered = T)

# Plot the data (top 50)
plt <- ggplot(q[1:50,], aes(x = ComplaintType, y = No.Complaints)) + 
            geom_bar(stat= 'identity') +
            theme_minimal() + theme(axis.text.x = element_text(angle = 45, hjust = 1))

# Convert to plotly object
py <- plotly()
py$ggplotly(plt, session='knitr')
```

<iframe height="600" id="igraph" scrolling="no" seamless="seamless"
				src="https://plot.ly/~rmdk/85" width="600" frameBorder="0"></iframe>

<br>

#### What is the most common complaint in each city?

How many cities are in the database?


```r
# dt[, unique(City)]

q <- data %>% select(City) %>% distinct() %>% summarise(Number.of.Cities = n())
head(q)
```

```
##   Number.of.Cities
## 1             1818
```

<br>

#### Yikes - let's just plot the 10 most complained about cities


```r
# dt[, (No.Complaints = .N), City]
#setkey(dt, No.Complaints)

q <- data %>% select(City) %>% group_by(City) %>% 
        summarise(No.Complaints = n()) %>%
        arrange(-No.Complaints)

head(q, 10)
```

```
##             City No.Complaints
## 1       BROOKLYN       2671085
## 2       NEW YORK       1692514
## 3          BRONX       1624292
## 4                       766378
## 5  STATEN ISLAND        437395
## 6        JAMAICA        147133
## 7       FLUSHING        117669
## 8        ASTORIA         90570
## 9        Jamaica         67083
## 10     RIDGEWOOD         66411
```

<br>

- Flushing and FLUSHING, Jamaica and JAMAICA... the complaints are case sensitive.


<br><br>

#### Perform case insensitive queries with GROUP BY with COLLATE NOCASE

- I decided to use `UPPER` to convert the CITY format.


```r
# dt[, CITY := toupper(City)][, (No.Complaints = .N), CITY]
#setkey(dt, No.Complaints)

# No clear way to do this using dplyr functions - default to regular SQL
q <- tbl(db, sql('SELECT UPPER(City) as "CITY", COUNT(*) as "No.Complaints"
                    FROM NYC_data 
                    GROUP BY "CITY" 
                    ORDER BY -"No.Complaints"
                    LIMIT 11'))

head(q, 10)
```

```
##             CITY No.Complaints
## 1       BROOKLYN       2671085
## 2       NEW YORK       1692514
## 3          BRONX       1624292
## 4                       766378
## 5  STATEN ISLAND        437395
## 6        JAMAICA        147133
## 7       FLUSHING        117669
## 8        ASTORIA         90570
## 9        JAMAICA         67083
## 10     RIDGEWOOD         66411
```

<br>

#### Complaint type count by city

- The plotly article loops through each city and runs an sql command. Simply grouping by two columns `City` and `Complaint` type accomplishes the same task. I reduce the number of items plotted here for display purposes. 



```r
# dt[, CITY := toupper(City)][, (No.Complaints = .N), .(ComplaintType, CITY)]
#setkey(dt, No.Complaints)

# No clear way to do this using dplyr functions - default to regular SQL
q <- tbl(db, sql('SELECT "Complaint Type" as "ComplaintType", UPPER(City) as "CITY", 
                    COUNT(*) as "No.Complaints"
                    FROM NYC_data 
                    GROUP BY CITY, ComplaintType
                    ORDER BY -"No.Complaints"'))


# Select only list of cities used in ploty article
q_f <- filter(q, CITY %in% c(
                        'FAR ROCKAWAY',
                        'FLUSHING',
                        'JAMAICA',
                        'STATEN ISLAND',
                        'BRONX',
                        'NEW YORK',
                        'BROOKLYN'
))

# Pull the data out of memory to plot it
q_f <- collect(q_f)

# Select a cutoff for the most popular complaints to plot
# dt[, sum(No.Complaints), ComplaintType][, .(ComplaintType)]
top_complaints <- q_f%>% group_by(ComplaintType) %>% 
                        summarise(n = sum(No.Complaints)) %>%
                        arrange(-n) 
# Top 15 complaints
q_f <- filter(q_f, ComplaintType %in% top_complaints$ComplaintType[1:15])

# Plot result
plt <- ggplot(q_f, aes(ComplaintType, No.Complaints, fill = CITY)) + 
            geom_bar(stat = 'identity') + 
            theme_minimal() + theme(axis.text.x = element_text(angle = 45, hjust = 1))

plt
```

![](r-code_files/figure-html/unnamed-chunk-18-1.png) 

```r
# plotly cannot handle this type of plot currently.
```

<br>

Now let's normalize these counts. This is super easy now that this data has been reduced into a dataframe.


```r
# dt[, No.Complaints_normalized := No.Complaints / sum(No.Complaints)]
q_f <- q_f %>% group_by(CITY) %>% 
                mutate(Normalized.Complaints = round((No.Complaints / sum(No.Complaints))*100, 2))

# Plot result
plt <- ggplot(q_f, aes(ComplaintType, Normalized.Complaints, fill = CITY)) + 
    geom_bar(stat = 'identity', position = 'dodge') + 
    theme_minimal() + theme(axis.text.x = element_text(angle = 45, hjust = 1)) + 
    ggtitle('Relative Number of Complaints by City')

plt
```

![](r-code_files/figure-html/unnamed-chunk-19-1.png) 

```r
# plotly cannot handle this type of plot currently.
```

- New York is loud.
- Staten Island is quite dirty, dark and soggy
- Bronx has great garbage collection.
- Flushing's muni meters are broken.

## Part 2 Time Series Operations

The data provided does not fit the standard date format for SQLite. We have a few options to remedy this:

1. Create a new column in the SQL database, and reinsert the data with the formatted date statements
2. Create a new table and INSERT the formatted date into the original column name
3. Generate a SELECT wrapper using dplyr that formats the dates. 

To illustrate using R, I will use dplyr. This is not necessarily the most optimal approach. Here I select substrings of the date fields, with `SUBSTR` and concatenate them with `||`.  


```r
data <- tbl(db, sql('SELECT Agency, "Complaint Type" AS ComplaintType, Descriptor, City,
                    SUBSTR("Created Date", 7, 4) || "-" ||
                    SUBSTR("Created Date", 4, 2) || "-" ||
                    SUBSTR("Created Date", 1, 2) || " " ||
                    SUBSTR("Created Date", 12, 2) || ":" ||
                    SUBSTR("Created Date", 15, 2) || ":" ||
                    SUBSTR("Created Date", 18, 2) as CreatedDate, 
                    
                    SUBSTR("Closed Date", 7, 4) || "-" ||
                    SUBSTR("Closed Date", 4, 2) || "-" ||
                    SUBSTR("Closed Date", 1, 2) || " " ||
                    SUBSTR("Closed Date", 12, 2) || ":" ||
                    SUBSTR("Closed Date", 15, 2) || ":" ||
                    SUBSTR("Closed Date", 18, 2) as ClosedDate
                    
            FROM NYC_data'))
```

<br>

#### Filter SQLite rows with timestamp strings: YYYY-MM-DD hh:mm:ss


```r
# dt[CreatedDate < '2014-11-26 23:47:00' & CreatedDate > '2014-09-16 23:45:00', 
#      .(ComplaintType, CreatedDate, City)]

q <- data %>% filter(CreatedDate < "2014-11-26 23:47:00",   CreatedDate > "2014-09-16 23:45:00") %>%
    select(ComplaintType, CreatedDate, City)

head(q)
```

```
##             ComplaintType         CreatedDate     City
## 1 Noise - Street/Sidewalk 2014-11-12 11:59:56    BRONX
## 2          Taxi Complaint 2014-11-12 11:59:40 BROOKLYN
## 3      Noise - Commercial 2014-11-12 11:58:53 BROOKLYN
## 4      Noise - Commercial 2014-11-12 11:58:26 NEW YORK
## 5 Noise - Street/Sidewalk 2014-11-12 11:58:14 NEW YORK
## 6      Noise - Commercial 2014-11-12 11:56:20    BRONX
```

<br>

#### Pull out the hour unit from timestamps with strftime

- See ?strftime for all`format` methods


```r
# dt[, hour := strftime('%H', CreatedDate), .(ComplaintType, CreatedDate, City)]

q <- data %>% mutate(hour = strftime('%H', CreatedDate)) %>% 
            select(ComplaintType, CreatedDate, City, hour)

head(q)
```

```
##             ComplaintType         CreatedDate     City hour
## 1 Noise - Street/Sidewalk 2015-11-04 02:13:04 BROOKLYN   02
## 2 Senior Center Complaint 2015-11-04 02:12:05 ELMHURST   02
## 3      Noise - Commercial 2015-11-04 02:11:46  JAMAICA   02
## 4 Noise - Street/Sidewalk 2015-11-04 02:11:02 BROOKLYN   02
## 5 Noise - Street/Sidewalk 2015-11-04 02:10:45 NEW YORK   02
## 6 Noise - Street/Sidewalk 2015-11-04 02:09:07 BROOKLYN   02
```

<br>

#### Count the number of complaints (rows) per hour with strftime , GROUP BY , and count(*) (n())


```r
# dt[, hour := strftime('%H', CreatedDate), .N , hour]

q <- data %>% mutate(hour = strftime('%H', CreatedDate)) %>% 
    group_by(hour) %>% summarise(Complaints.per.Hour = n())

# Collect the data into memory
q <- collect(q)

plt <- ggplot(na.omit(q), aes(hour, Complaints.per.Hour)) + 
        geom_bar(stat='identity') + theme_minimal()

# Convert to plotly object
py <- plotly()
py$ggplotly(plt, session='knitr')
```

<iframe height="600" id="igraph" scrolling="no" seamless="seamless"
				src="https://plot.ly/~rmdk/86" width="600" frameBorder="0"></iframe>

<br>

#### Filter noise complaints by hour (might be easier to use `LIKE` operator in raw SQL)


```r
# dt[grepl(ComplaintType, 'Noise'), hour := strftime('%H', CreatedDate), .N , hour]

q <- data %>% filter(ComplaintType %in% c( "Noise",
                                           "Noise - Street/Sidewalk", 
                                           "Noise - Commercial", 
                                           "Noise - Vehicle", 
                                           "Noise - Park", 
                                           "Noise - House of Worship", 
                                           "Noise - Helicopter", 
                                           "Collection Truck Noise"))  %>%
    mutate(hour = strftime('%H', CreatedDate)) %>% 
    group_by(hour) %>% summarise(Complaints.per.Hour = n())

# Collect the data into memory
q <- collect(q)
# omit NA
plt <- ggplot(na.omit(q), aes(hour, Complaints.per.Hour)) + 
            geom_bar(stat='identity') + theme_minimal()

# Convert to plotly object
py <- plotly()
py$ggplotly(plt, session='knitr')
```

<iframe height="600" id="igraph" scrolling="no" seamless="seamless"
				src="https://plot.ly/~rmdk/87" width="600" frameBorder="0"></iframe>

<br>

#### Segregate complaints by hour

- This can be written more succinctly than the plotly article using two fields in the `GROUP_BY()` statement. 


```r
# dt[grepl(ComplaintType, 'Noise'), hour := strftime('%H', CreatedDate), .N , hour]
# setkey(dt, Complaints.per.Hour)
# dt[, tail(.SD, 2), hour]

q <- data %>% 
    mutate(hour = strftime('%H', CreatedDate)) %>% 
    group_by(hour, ComplaintType) %>% summarise(Complaints.per.Hour = n())

# Collect the data into memory
q <- collect(q)
# omit NA
q <- na.omit(q)

# Grab the 2 most common complaints for that hour
# Top 6 is way too many colors
q_hr <-q %>%
      group_by(hour) %>%
      top_n(n = 2, wt = Complaints.per.Hour)

# Plot (something appears to be off with midnight in my dataset)
plt <- ggplot(q_hr[q_hr$hour != '12',], aes(hour, Complaints.per.Hour, fill = ComplaintType)) + 
            geom_bar(stat='identity') + theme_minimal()

# Convert to plotly object
py <- plotly()
py$ggplotly(plt, session='knitr')
```

<br>

### Aggregate Time Series

#### First, create a new column with timestamps rounded to the previous 15 minute interval


```r
# Using lubridate::new_period()
# dt[, interval := CreatedDate - new_period(900, 'seconds')][, .(CreatedDate, interval)]

q <- data %>% 
     mutate(interval = sql("datetime((strftime('%s', CreatedDate) / 900) * 900, 'unixepoch')")) %>%                     
     select(CreatedDate, interval)

head(q, 10)
```

```
##            CreatedDate            interval
## 1  2015-11-04 02:13:04 2015-11-04 02:00:00
## 2  2015-11-04 02:12:05 2015-11-04 02:00:00
## 3  2015-11-04 02:11:46 2015-11-04 02:00:00
## 4  2015-11-04 02:11:02 2015-11-04 02:00:00
## 5  2015-11-04 02:10:45 2015-11-04 02:00:00
## 6  2015-11-04 02:09:07 2015-11-04 02:00:00
## 7  2015-11-04 02:05:47 2015-11-04 02:00:00
## 8  2015-11-04 02:03:43 2015-11-04 02:00:00
## 9  2015-11-04 02:03:29 2015-11-04 02:00:00
## 10 2015-11-04 02:02:17 2015-11-04 02:00:00
```

<br>

#### Then, GROUP BY that interval and COUNT(*)


```r
# Using lubridate::new_period()
# dt[, interval := CreatedDate - new_period(900, 'seconds')][, .N, interval]

q <- data %>% 
     mutate(interval = sql("datetime((strftime('%s', CreatedDate) / 900) * 900, 'unixepoch')")) %>%  
     group_by(interval) %>%
     summarise(Complaints.per.Interval = n()) %>% filter(!is.na(Complaints.per.Interval))

head(q, 10)
```

```
##               interval Complaints.per.Interval
## 1                 <NA>                 5447756
## 2  2003-01-03 01:00:00                       1
## 3  2003-01-03 03:15:00                       2
## 4  2003-01-03 06:30:00                       2
## 5  2003-01-03 06:45:00                       1
## 6  2003-01-03 07:30:00                       1
## 7  2003-01-03 09:15:00                       1
## 8  2003-01-03 09:30:00                       1
## 9  2003-01-03 09:45:00                       1
## 10 2003-01-03 11:00:00                       1
```

```r
# Pull data into memory
q <- collect(q %>% filter(strftime('%Y', interval) == '2003'))
```

<br>

#### Plot the results for 2003


```r
# Convert to proper datetime object
q$interval <- strptime(q$interval, '%Y-%m-%d %H:%M')

plt <- ggplot(q, aes(x=interval, y=Complaints.per.Interval)) + 
            geom_bar(stat='identity') +
            theme_minimal() 

# Convert to plotly object
py <- plotly()
py$ggplotly(plt, session='knitr')
```

<iframe height="600" id="igraph" scrolling="no" seamless="seamless"
				src="https://plot.ly/~rmdk/89" width="600" frameBorder="0"></iframe>

<br>

#### Same plot by hours


```r
# Using lubridate::new_period()
# dt[, interval := CreatedDate - new_period(86400, 'seconds')][, .N, interval]

q <- data %>% 
     mutate(interval = sql("datetime((strftime('%s', CreatedDate) / 86400) * 86400, 'unixepoch')")) %>%  
     group_by(interval) %>%
     summarise(Complaints.per.Interval = n()) %>% filter(!is.na(Complaints.per.Interval))

# Collect into memory
q <- collect(q)
# Convert to proper datetime object
q$interval <- strptime(q$interval, '%Y-%m-%d %H:%M')

plt <- ggplot(q, aes(x=interval, y=Complaints.per.Interval)) + 
        geom_bar(stat='identity') +
        theme_minimal()

# Convert to plotly object
py <- plotly()
py$ggplotly(plt, session='knitr')
```

<iframe height="600" id="igraph" scrolling="no" seamless="seamless"
				src="https://plot.ly/~rmdk/90" width="600" frameBorder="0"></iframe>

## Contact me

#### [twitter](https://twitter.com/ryanmdk) | email.ryan.kelly@gmail.com | [github](https://github.com/RMDK/bigish-data-in-R)
