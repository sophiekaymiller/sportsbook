# Sports Betting Lines


```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## This function is used to scrape the historical Vegas Lines for NFL games


```{r lib, message=FALSE, warning=FALSE}
# Load libraries
library(lubridate)
library(tidyverse)
library(rvest)
library(magrittr)
library(knitr)
```

## Get Lines Function

```{r function}

GetLines <- function(sport = "NBA", year, type = "both") {
  
  options(stringsAsFactors = FALSE)
  sport <- tolower(sport)
  
  # Get team urls and names
  url <- paste0("http://www.covers.com/pageLoader/pageLoader.aspx?page=/data/", sport, "/teams/teams.html")
  html <- suppressWarnings(readLines(url))
  links <- html[grep(paste0("/data/", sport, "/teams/team[0-9]"), html)]
  links <- unique(gsub(paste0('.*/data/', sport,'/teams/([^\"]*)\"[ ]*>([^<]*).*'), '\\1,\\2', links))
  links <- strsplit(links, ',')
  
  # Create URL stems for getting team records
  url.base <- paste(gsub("SPORT", sport, "http://www.covers.com/pageLoader/pageLoader.aspx?page=/data/SPORT/teams/pastresults/"), year - 1, '-', year, '/', sep = '')
  full.lines <- data.frame()
  
  # Put all team records in one data frame
  for (i in 1:length(links)) {
    
    url <- paste(url.base, links[[i]][1], sep = "")
    tables <- read_html(url) %>% html_table(header = TRUE, fill = TRUE) 
    
    # Get appropriate data: regular season, playoffs, or both
    if (type == 'regular season') {
      
      if (length(tables) == 0) {  # If the team didn't exist at the time, make an empty df
        temp <- data.frame(matrix(nrow = 1, ncol = 6, data = 0))
        temp <- temp[-1, ]
      } else {
        if (sport == 'nfl') {
          temp <- tables[[length(tables) - 1]]  # Regular season is always the second to last table
        } else if (sport == 'nba') {
          temp <- tables[[length(tables)]]      # Regular season is always the last table
        }
      }
      
    } else if (type == 'playoffs') {
      
      if (length(tables) == 1) {  # If the team didn't make the playoffs, make an empty df
        temp <- data.frame(matrix(nrow = 1, ncol = 6, data = 0))
        temp <- temp[-1, ]
      } else {
        temp <- tables[[1]]
      }
      
    } else {
      
      if (length(tables) == 0) {  # If the team didn't exist at the time, make an empty df
        temp <- data.frame(matrix(nrow = 1, ncol = 6, data = 0))
        temp <- temp[-1, ]
      } else {
        if (sport == 'nfl') {
          if (length(tables) == 2) {
            temp <- tables[[1]]
          } else {
            temp <- rbind(tables[[1]], tables[[2]])
          }
        } else if (sport == 'nba') {
          if (length(tables) == 1) {
            temp <- tables[[1]]
          } else {
            temp <- rbind(tables[[1]], tables[[2]])
          }
        }
      }
      
    }
    
    colnames(temp) <- c("date", "away.team", "score", "type", "home.line", "over.under")
    
    if (nrow(temp) > 0) {   # Add home team if the df has any info (we didn't ask for playoffs from a team that didn't make it)
      temp$home.team <- links[[i]][2]
    }
    
    try(full.lines <- rbind(full.lines, temp), silent = FALSE)
  }
  
  # Extra formatting for NFL df
  if (sport == 'nfl') {
    full.lines <- full.lines %>% filter(score != "BYE") 
    full.lines$date <- gsub("[A-Za-z ]*", "", full.lines$date)
  }
  
  # Format date
  full.lines$date <- mdy(full.lines$date)
  
  # Remove away games (the odds exist for home team already)
  full.lines <- full.lines[-grep('@', full.lines$away.team), ]
  
  # Create home score and away score columns
  full.lines$home.score <- as.numeric(gsub('[^0-9]*([0-9]+)-([0-9]+)[^0-9]*', '\\1', full.lines$score))
  full.lines$away.score <- as.numeric(gsub('[^0-9]*([0-9]+)-([0-9]+)[^0-9]*', '\\2', full.lines$score))
  
  # Fix line and over/under
  full.lines$home.line <- as.numeric(gsub('[^0-9\\.-]', '', full.lines$home.line))
  full.lines$over.under <- as.numeric(gsub('[^0-9\\.-]', '', full.lines$over.under))
  
  # Reorder columns, sort by date, and return
  full.lines <- full.lines[, c('date', 'type', 'home.team', 'away.team', 'home.score', 'away.score', 'home.line', 'over.under')]
  full.lines <- full.lines[order(full.lines$date), ]
  return(full.lines)
}
```


## This is how to use the function

#### Parameters:

* **sport:** NFL or NBA
* **year:** season. (2012 would return the data from the 2011-2012 season)  
* **type:** 
    + "regular season"
    + "playoffs"
    + "both"


```{r}
# example
NFL.2017 <- GetLines(sport = "NFL", year = 2017, type = "both")
NFL.2018 <- GetLines(sport = "NFL", year = 2018, type = "both")
```

## Generate Table

```{r}
kable(NFL.2017)
```

## Combine multiple seasons of data together.  

```{r}

GetLinesRange <- function(sport = "NBA", year.start, year.end, type = "both") {
  lines <- data.frame()
  
  for (year in year.start:year.end) {
    temp <- GetLines(sport, year, type)
    temp$season <- year
    lines <- rbind(lines, temp)
  }
  
  return(lines)
}
```

